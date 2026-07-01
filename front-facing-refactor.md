# WESL Front-Facing API Refactor — Design & Implementation Plan

## Context

`wesl` (crates/wesl) compiles WESL → WGSL through a fixed, mostly-private pipeline:
resolve → parse → condcomp → import resolution → mangle → assemble → validate → lower →
strip → stringify. The only public entry is the `Wesl<R: Resolver>` facade plus two free
functions (`compile`, `compile_sourcemap`, lib.rs:904/924). This makes four things hard:

1. **Custom AST transforms** — the only injection point is abusing `Preprocessor` (a
   resolver wrapper); that's how condcomp itself is wired in (lib.rs:844). No
   whole-program hook exists at all.
2. **Invoking steps individually** — `compile_pre_assembly`, `Resolutions::{mangle,assemble}`,
   `compile_post_assembly` are private; parse-only / lower-only / stringify-an-AST
   pipelines can't be built from parts.
3. **Async loading** — `Resolver::resolve_source` is sync; JS/wasm Promise loaders and
   engine asset pipelines can't feed the compiler.
4. **Sourcemapping** — `BasicSourceMap` maps mangled name → (module, name) only; no
   spans/lines, no way to emit origin comments into output WGSL. Root (unmangled) decls
   are missing entirely (error.rs:243 `XXX`).

Also: `compile_sourcemap` duplicates `compile`; CLI/wesl-web/wesl-c all hand-roll the
same setup; `add_package` only exists on `Wesl<StandardResolver>` so virtual-resolver
users (wasm) can't use packages via the facade; the type-changing `set_custom_resolver`
forces wesl-cli to duplicate `.compile()` call sites and prevents wesl-c from storing a
configured compiler behind FFI.

**Requirements:** concise default path (~3 lines) · pluggable + individually invocable
pipeline steps · sync **and** async module loading · declaration provenance (file+line,
origin comments in output).

## Facts that constrain the design (verified in code)

- Lazy import resolution (`import.rs:271`) interleaves load → parse → import-flattening →
  usage-discovery per module through deep mutual recursion (`resolve_decl` ⇄ `resolve_ty`
  ⇄ `load_module`), resolver calls buried inside. Condcomp deletes import statements, so
  **per-module transforms must run at load time** — a whole-graph "condcomp step" is
  impossible under lazy loading.
- `strip_except` keeps decls with `Ident::use_count() > 1` (strip.rs:12); `Ident` is
  `Arc<RwLock<String>>` (wgsl-parse/syntax.rs:53). Two consequences:
  **(a)** the module graph must be dropped before stripping (today: `into_module_order`
  at lib.rs:914) — so the granular "assemble" step must **consume** the graph;
  **(b)** a sourcemap holding strong `Ident` clones would inflate every use-count and
  silently disable stripping — provenance must be keyed by **weak** handles.
- Spans (`Spanned<T>`, byte offsets into the original module source) survive `assemble`
  unchanged; per-decl `Display` exists (`GlobalDeclaration: Display`,
  syntax_display.rs:161) — origin comments need **zero wgsl-parse changes**. The AST has
  no comment nodes (lexer skips them); that stays true.
- The **only I/O in the whole pipeline is module loading**; everything after resolution
  is pure CPU.
- `Ident` is `Arc`-based → the AST is `Send`; only `Resolutions`'
  `Rc<RefCell<Module>>` is `!Send` (internal, changeable later).
- MSRV 1.96 / edition 2024: AFIT stable (not dyn-compatible); `Waker::noop` stable.
- `CompileOptions` derives `Clone + PartialEq + Eq` (lib.rs:67) → boxed passes can't
  live in it. `Error` is `Clone` with a `Custom(String)` variant (error.rs:20/41) →
  pass errors are `crate::Error`, not `Box<dyn Error>`.
- build.rs consumers must stay sync + zero-dep; wasm can't block; wesl-c stays sync.

---

# The design: three layers, AST as the currency

`TranslationUnit` (the public wgsl-parse AST) is the visible currency between all steps,
so users can always drop to raw AST manipulation. Three layers sit on top of it:

```
Layer 2  Compiler          facade: config + tools + hooks; compile()/compile_async()
Layer 1  Compilation       pipeline state (the module graph); load → validate → mangle → assemble
Layer 0  wesl::passes      free functions on &mut TranslationUnit (condcomp, lower, strip…)
```

`Compiler::compile()` is *implemented from the public Layer-1/0 API* — guaranteeing the
granular API can express everything the facade does.

## Layer 0 — `wesl::passes`: single-unit transforms as free functions

New module `crates/wesl/src/passes.rs`, mostly promotions of existing private fns:

```rust
pub mod passes {
    pub fn condcomp(unit: &mut TranslationUnit, features: &Features) -> Result<(), Error>; // was condcomp::run (private)
    pub fn mangle_module(unit: &mut TranslationUnit, path: &ModulePath, mangler: &dyn Mangler); // was import::mangle_decls
    pub fn lower(unit: &mut TranslationUnit) -> Result<(), Error>;                         // re-export of existing pub lower
    pub fn strip_unused(unit: &mut TranslationUnit, keep: &HashSet<Ident>);                // was strip::strip_except
    pub fn validate_wesl(unit: &TranslationUnit) -> Result<(), Diagnostic<Error>>;         // re-export
    pub fn validate_wgsl(unit: &TranslationUnit) -> Result<(), Diagnostic<Error>>;         // re-export
    #[cfg(feature = "generics")]
    pub fn monomorphize(unit: &mut TranslationUnit) -> Result<(), Error>;                  // generate_variants + replace_calls
}
```

Parsing is already granular (`str::parse::<TranslationUnit>()`), stringification is
`Display`. Existing root exports (`lower`, `validate_*`) remain for compat.

## Layer 1 — `Compilation`: the pipeline-state type

New file `crates/wesl/src/compilation.rs`, wrapping the pub(crate) `Resolutions` graph:

```rust
pub struct Compilation {
    resolutions: import::Resolutions,   // module graph (stays pub(crate))
    keep: HashSet<Ident>,               // strip roots; also lazy-load roots
    lazy: bool,                         // was the graph loaded lazily (drives assemble mode)
    sourcemap: Option<BasicSourceMap>,  // accumulating (sources + provenance)
}

impl Compilation {
    /// Load + link a module graph. THE ONLY I/O STEP in the pipeline.
    /// Consumes options: imports, lazy, strip, keep, keep_root, condcomp, features, sourcemap.
    pub fn load(root: &ModulePath, resolver: &impl Resolver, options: &CompileOptions) -> Result<Self, Error>;
    pub async fn load_async(root: &ModulePath, resolver: &impl AsyncResolver, options: &CompileOptions) -> Result<Self, Error>;
    /// Wrap one pre-parsed module; no import resolution (single-file mode).
    pub fn from_module(path: ModulePath, unit: TranslationUnit) -> Self;

    pub fn root_path(&self) -> &ModulePath;
    pub fn module_paths(&self) -> impl Iterator<Item = &ModulePath>;
    pub fn keep_idents(&self) -> &HashSet<Ident>;
    pub fn sourcemap(&self) -> Option<&BasicSourceMap>;

    pub fn validate(&self) -> Result<(), Error>;          // validate_wesl per module, module-path context
    pub fn mangle(&mut self, mangler: &dyn Mangler);      // external modules only (spec default)
    pub fn mangle_all(&mut self, mangler: &dyn Mangler);  // incl. root (today's mangle_root: true)

    /// Merge into one unit. CONSUMES self on purpose: strip/DCE relies on
    /// `Ident::use_count()`, which must not count refs still held by the graph.
    pub fn assemble(self) -> CompileResult;               // also finalizes sourcemap output-name index
}
```

Design points:
- **`assemble(self)` consuming is the load-bearing signature** — it encodes the one
  silently-corrupting ordering constraint in the type system. No type-state ladder needed.
- No mutable module access on `Compilation`: usage analysis/retargeting happen during
  `load`; the two safe mutation points are per-module passes (before analysis) and the
  assembled unit (after). Documented constraint. (A guarded `ResolutionPass`/`ModuleSet`
  pre-mangle stage is designed but deferred — under lazy+strip, decls added there are
  silently dropped by `assemble`'s used-filter unless registered in `used_idents`.)
- The keep set is computed at `load` (it triple-serves: lazy-discovery roots, usage
  marking, DCE roots — lib.rs:809 `keep_idents`), exposed via `keep_idents()`, carried
  into `CompileResult.keep`.

`CompileResult` keeps its name and pub fields (preserves `eval`/`exec`/`build_artifact`
and every consumer), grows two members:

```rust
pub struct CompileResult {
    pub syntax: TranslationUnit,
    pub sourcemap: Option<BasicSourceMap>,
    pub modules: Vec<ModulePath>,
    pub keep: HashSet<Ident>,           // NEW: strip roots, so manual pipelines can strip
}
impl CompileResult {
    pub fn strip(&mut self);            // sugar for passes::strip_unused(&mut self.syntax, &self.keep)
    pub fn to_string_with_origins(&self) -> String;                                        // NEW (sourcemap v2)
    pub fn to_string_annotated(&self, f: impl Fn(&DeclLocation) -> Option<String>) -> String; // NEW
    // unchanged: Display, write_to_file, write_artifact, eval(), exec()
}
```

## Layer 2 — `Compiler`: the facade

New file `crates/wesl/src/compiler.rs`. **Defaulted type parameter** resolves the
facade-genericity tension (non-generic ergonomics + FFI storability for the common case;
type-safe sync/async split for the rest — an AFIT `AsyncResolver` can't go in a
`Box<dyn>`, and wasm resolvers are `!Send`, so a fully non-generic facade can't carry them):

```rust
pub struct Compiler<R = Box<dyn Resolver + Send + Sync>> {
    options: CompileOptions,
    resolver: R,
    mangler: Box<dyn Mangler + Send + Sync>,
    packages: Vec<&'static CodegenPkg>,              // moved off StandardResolver
    constants: HashMap<String, LiteralInstance>,     // moved off StandardResolver
    passes: Passes,                                  // hook registry (below)
}

impl Compiler {                                      // the default (boxed) type
    pub fn new(base: impl AsRef<Path>) -> Self;      // FileResolver + spec defaults
    pub fn new_experimental(base: impl AsRef<Path>) -> Self;
    pub fn new_barebones() -> Self;
    /// Swap the resolver WITHOUT changing the compiler's type (fixes wesl-cli's
    /// duplicated match arms; lets wesl-c store one configured Compiler across calls).
    pub fn set_resolver(&mut self, r: impl Resolver + Send + Sync + 'static) -> &mut Self;
}

impl<R> Compiler<R> {
    pub fn with_resolver(resolver: R) -> Self;       // unboxed / !Send / async resolvers
    // config (all &mut Self -> &mut Self, chainable):
    pub fn set_options(&mut self, options: CompileOptions) -> &mut Self;
    pub fn options_mut(&mut self) -> &mut CompileOptions;   // replaces the long tail of use_* forwarders
    pub fn set_mangler(&mut self, kind: ManglerKind) -> &mut Self;
    pub fn set_custom_mangler(&mut self, m: impl Mangler + Send + Sync + 'static) -> &mut Self;
    pub fn add_package(&mut self, pkg: &'static CodegenPkg) -> &mut Self;   // now works with ANY resolver
    pub fn add_constant(&mut self, name: impl ToString, value: LiteralInstance) -> &mut Self;
    pub fn set_feature(&mut self, feat: &str, val: impl Into<Feature>) -> &mut Self;
    // kept sugar: use_sourcemap/use_imports/use_condcomp/use_stripping/use_lower/
    //             keep_declarations/set_features/add_packages/add_constants/...
    // hooks:
    pub fn add_module_pass(&mut self, pass: impl ModulePass + 'static) -> &mut Self;
    pub fn add_program_pass(&mut self, stage: Stage, pass: impl ProgramPass + 'static) -> &mut Self;
}

impl<R: Resolver> Compiler<R> {
    pub fn compile(&self, root: &ModulePath) -> Result<CompileResult, Error>;
    pub fn load(&self, root: &ModulePath) -> Result<Compilation, Error>;    // granular entry w/ passes applied
    pub fn build_artifact(&self, root: &ModulePath, artifact_name: &str);   // unchanged behavior
}

impl<R: AsyncResolver> Compiler<R> {
    pub async fn compile_async(&self, root: &ModulePath) -> Result<CompileResult, Error>;
    pub async fn load_async(&self, root: &ModulePath) -> Result<Compilation, Error>;
}
```

- Packages/constants compose into an *effective resolver* at load time
  (constants module + `PkgResolver` + user resolver) — fixes `add_package` being
  `StandardResolver`-only today (lib.rs:288). `StandardResolver` stays public.
- `CompileOptions` stays ONE plain struct (wesl-web/wesl-c build it wholesale from
  JSON/FFI; per-stage structs would multiply their mapping code). One addition:
  `sourcemap: bool` (default true), absorbing the facade's separate `use_sourcemap` field.
- `compile()` wraps everything in the error-decoration path (`with_output`, `unmangle`,
  sourcemap context) that `compile_sourcemap` does today (lib.rs:938-957), eliminating
  the compile/compile_sourcemap duplication.

`compile()` body — the facade is a composition of public steps:

```rust
let mut comp = self.load(root)?;                      // I/O; per-module: parse → condcomp → module passes
if opts.validate { comp.validate()?; }
if opts.mangle_root { comp.mangle_all(&*self.mangler) } else { comp.mangle(&*self.mangler) }
let mut result = comp.assemble();                     // consumes graph; finalizes name index
for p in &self.passes.post_assembly { p.run(&mut result.syntax, &ctx)?; }   // Stage::PostAssembly
#[cfg(feature = "generics")] if opts.generics { passes::monomorphize(&mut result.syntax)?; }
if opts.validate { passes::validate_wgsl(&result.syntax)?; }
if opts.lower    { passes::lower(&mut result.syntax)?; }
if opts.strip    { result.strip(); }
for p in &self.passes.finalize { p.run(&mut result.syntax, &ctx)?; }        // Stage::Final
Ok(result)
```

## Hook system (custom AST transformations)

New file `crates/wesl/src/pass.rs` (types re-exported at root). Traits with blanket
closure impls — object-safe, nameable in diagnostics, stateful passes possible:

```rust
pub trait ModulePass: Send + Sync {
    fn name(&self) -> &str { "<module pass>" }
    fn run(&self, module: &mut TranslationUnit, ctx: &ModuleCtx) -> Result<(), Error>;
}
pub trait ProgramPass: Send + Sync {
    fn name(&self) -> &str { "<program pass>" }
    fn run(&self, unit: &mut TranslationUnit, ctx: &ProgramCtx) -> Result<(), Error>;
}
impl<F: Fn(&mut TranslationUnit, &ModuleCtx) -> Result<(), Error> + Send + Sync> ModulePass for F { ... }
impl<F: Fn(&mut TranslationUnit, &ProgramCtx) -> Result<(), Error> + Send + Sync> ProgramPass for F { ... }

pub struct ModuleCtx<'a> { /* private */ }   // .path() .root() .is_root() .options() .features()
pub struct ProgramCtx<'a> { /* private */ }  // .stage() .root() .options()

#[non_exhaustive]
pub enum Stage { PostAssembly, /* merged+mangled, before validate/lower/strip */
                 Final /* after strip, right before stringification; not re-validated */ }

#[derive(Default)]
pub struct Passes {           // lives on Compiler, NOT CompileOptions (which derives Eq)
    pub module: Vec<Box<dyn ModulePass>>,
    pub post_assembly: Vec<Box<dyn ProgramPass>>,
    pub finalize: Vec<Box<dyn ProgramPass>>,
}
```

Hook points and why exactly these:

| Stage | Runs | Why here |
|---|---|---|
| **ModuleLoad** (per module) | inside loading, after parse + builtin condcomp, before ident-map/retarget/usage-discovery; root included | the only point compatible with lazy resolution; additions/renames/new imports are fully integrated by all later steps (incl. `keep_idents` for the root) |
| **PostAssembly** | on the merged, mangled unit, before validate/lower/strip | first "whole program" moment; pass mistakes still get validated/stripped |
| **Final** | after strip, before stringify | post-processing that must not be altered by lower/strip; not re-validated (documented) |

Contract: within a stage, registration order; condcomp runs before user module passes.
Pass errors get module path + source + pass name attached (reproducing `Preprocessor`'s
diagnostic decoration at resolve.rs:253-261, minus its `unwrap`). `Error::custom(msg)`
helper added for user pass errors.

Internally, the `Preprocessor`-wrapping + `Box<dyn Resolver>` branch in
`compile_pre_assembly` (lib.rs:844-851) is replaced by an internal loader that carries:
the resolver, the module-pass chain (builtin condcomp first), and an optional sourcemap
recording slot. Public `Preprocessor` survives un-deprecated as user-facing resolver
middleware (also the recipe for per-module passes with bare `Compilation::load`).

## Sync/async: one pipeline, two entry points

**`AsyncResolver` (AFIT, no `Send` bounds), `AsAsync` adapter, async core, no-op-waker
sync driver. Zero new dependencies; no cargo feature.**

```rust
pub trait AsyncResolver {
    async fn resolve_source<'a>(&'a self, path: &ModulePath) -> Result<Cow<'a, str>, ResolveError>;
    async fn resolve_module(&self, path: &ModulePath) -> Result<TranslationUnit, ResolveError> { /* default: parse resolve_source, same as sync */ }
    fn display_name(&self, _: &ModulePath) -> Option<String> { None }   // metadata stays sync
    fn fs_path(&self, _: &ModulePath) -> Option<PathBuf> { None }
}
impl<T: AsyncResolver + ?Sized> AsyncResolver for &T { ... }
impl<T: AsyncResolver + ?Sized> AsyncResolver for Box<T> { ... }

/// Adapter: any sync Resolver is an AsyncResolver whose futures NEVER suspend
/// (bodies contain zero `.await`). NOT a blanket impl — that collides with
/// `impl Resolver for &T / Box<T>` coherence.
pub struct AsAsync<R>(pub R);
impl<R: Resolver> AsyncResolver for AsAsync<R> { ... }
```

- The pipeline core becomes `async fn` (written once). Sync entries drive it with a
  private ~15-line `block_on_ready` (`Waker::noop`): sound because sync-path futures
  complete in exactly one poll by construction — `.await` only returns `Pending` if an
  inner future does, and `AsAsync` bodies have no awaits. Misuse is a **type error**
  (sync API only accepts `impl Resolver`); the `Pending` arm is `unreachable!` with a
  message naming the invariant. Works on wasm (never parks) — unlike pollster.
- **Async recursion**: `resolve_decl` becomes fn-level `Box::pin`; `resolve_ty` boxes
  only its self-recursive call site; hot leaf calls allocate nothing. Bodies stay
  line-for-line identical (every `load_module(...)?` → `.await?`), preserving DFS module
  order (user-visible output order) and innermost-error attribution.
- **RefCell audit**: `Ref` borrows held across awaits are sound (future is local, never
  aliased; `borrow_mut` only on freshly-created modules or after resolution — position-
  identical to today's sync code). Compile future is `!Send` (internal `Rc`) — fine for
  wasm (`spawn_local`)/LocalSet; documented; non-breaking upgrade path to `Send` later
  by migrating `Resolutions` to index-based storage (`Ident` is already `Arc`).
- **Rejected**: blanket sync→async impl (coherence), sans-io state machine (forces
  owned-frame rewrite of the "XXX: quite messy" recursion; risks changing output order
  and error attribution — `async fn` generates that state machine from unchanged code),
  maybe-async macros (doc/debug pain), pollster (dep on build.rs path; converts a type
  error into a runtime hazard).
- **wesl-web**: keep sync `run`; add `JsResolver` wrapping a JS `(path) => Promise<string>`
  via `js_sys::Function` + `wasm_bindgen_futures::JsFuture`, and `pub async fn run_async`.
  New dep `wasm-bindgen-futures` in wesl-web only. **wesl-c / build.rs: zero changes.**

## Sourcemap v2: provenance + annotated output

**Keying insight:** name keys break under renames and miss unmangled root decls;
index arrays break under strip/lower `swap_remove`; strong `Ident` keys disable
stripping (use_count). The stable identity is a **weak handle to the `Ident` Arc**.

wgsl-parse (additive, ~30 lines): `Ident::downgrade() -> WeakIdent`;
`WeakIdent(Weak<RwLock<String>>)` with pointer-based Eq/Hash (consistent with `Ident`'s
hash; sound: live weak count pins the allocation → no address reuse → no false matches).
Does not affect `use_count`.

wesl sourcemap.rs (public API preserved; internals grow):

```rust
pub struct DeclProvenance { pub path: ModulePath, pub name: String /* pre-mangle */, pub span: Option<Span> }
pub struct DeclLocation<'a> { pub path: &'a ModulePath, pub name: &'a str,
                              pub file: Option<&'a str>,   // Resolver::display_name (FileResolver → real path)
                              pub line: Option<u32>, pub col: Option<u32> }  // 1-based, from stored sources

impl BasicSourceMap {
    // existing methods unchanged; internally: decls: Vec<DeclProvenance>,
    // by_name: HashMap<String, usize>, by_ident: HashMap<WeakIdent, usize>, sources (as today)
    pub fn provenance_of(&self, ident: &Ident) -> Option<&DeclProvenance>;
    pub fn provenance_by_name(&self, name: &str) -> Option<&DeclProvenance>;
    pub fn decl_location(&self, ident: &Ident) -> Option<DeclLocation<'_>>;
}
// SourceMap trait: gains defaulted get_provenance/get_provenance_by_name (non-breaking)
```

Recording (all internal): module sources+display names at load (today's
`SourceMapper::resolve_source` behavior, moved into the loader slot);
`record_provenance` over all modules at end of `load` — **pre-mangle**, capturing
(WeakIdent, path, original name, decl span); `index_output_names()` at `assemble` —
**post-mangle**, upgrading weak idents to map final output names (covers unmangled root
decls, fixing error.rs:243). The `SourceMapper` resolver+mangler proxy becomes
unnecessary internally (kept public for compat).

Robustness: renames (user passes, lower's alias inlining) — WeakIdent unaffected;
removals — stale entries never consulted (annotation walks the final decl list);
synthesized decls — no provenance → no comment (uniform rule). Regression test:
stripping still strips with sourcemap enabled.

Annotated output (new `crates/wesl/src/output.rs`): a writer mirroring
`TranslationUnit::fmt` byte-for-byte, prepending a comment per decl with provenance:

```wgsl
// package::util::rand (src/shaders/util.wesl:3)
fn package_util_rand() -> f32 { ... }
```

`to_string_annotated(|_| None) == to_string()` is a test invariant. Rejected: comment
nodes in wgsl-parse (invasive: lexer callbacks, grammar, Eq/serde/tokrepr on every node —
and origin annotation doesn't need them).

Error-diagnostic wins: root-decl names now resolve in `with_sourcemap`/`unmangle`
(CHANGELOG note: error text shows qualified names); decl-span fallback gives snippet-less
errors a caret at the declaration header; `decl_location` enables mapping
naga/driver errors on emitted WGSL back to WESL files.

---

# The four tasks, as they'll look

**Default path (unchanged conciseness):**
```rust
let wgsl = Compiler::new("src/shaders")
    .compile(&"package::main".parse()?)?
    .to_string();
```

**1. Custom AST transformations:**
```rust
let mut compiler = Compiler::new("src/shaders");
compiler.add_module_pass(|m: &mut TranslationUnit, ctx: &ModuleCtx| {
    if ctx.is_root() {
        let decl: GlobalDeclaration = "const DEBUG: bool = true;".parse()?;
        m.global_declarations.push(decl.into());
    }
    Ok(())
});
compiler.add_program_pass(Stage::Final, |u: &mut TranslationUnit, _: &ProgramCtx| {
    for mut ep in u.entry_points().cloned().collect::<Vec<_>>() {
        let name = format!("dbg_{}", ep.name());  // read guard dropped before rename
        ep.rename(name);                          // propagates to all references
    }
    Ok(())
});
```

**2. Async loading:**
```rust
struct HttpResolver { base: Url, client: Client }
impl AsyncResolver for HttpResolver {
    async fn resolve_source<'a>(&'a self, path: &ModulePath) -> Result<Cow<'a, str>, ResolveError> {
        let url = self.url_for(path);
        self.client.get(url).send().await?.text().await
            .map(Cow::Owned).map_err(|e| ResolveError::ModuleNotFound(path.clone(), e.to_string()))
    }
}
let result = Compiler::with_resolver(HttpResolver::new(base))
    .compile_async(&"package::main".parse()?).await?;
```
(wesl-web ships `JsResolver` + `run_async` so JS supplies `(path) => Promise<string>`.)

**3. Sourcemapped output with origin comments:**
```rust
let result = Compiler::new("src/shaders").compile(&root)?;   // sourcemap on by default
println!("{}", result.to_string_with_origins());
// custom format:
result.to_string_annotated(|loc| Some(format!("/* {}:{} */", loc.file.unwrap_or("?"), loc.line.unwrap_or(0))));
```

**4. Build-your-own pipeline of tasks:**
```rust
// single file, no imports: parse → condcomp → lower → stringify
let mut unit: TranslationUnit = std::fs::read_to_string("shader.wesl")?.parse()?;
passes::condcomp(&mut unit, &features)?;
passes::lower(&mut unit)?;
let wgsl = unit.to_string();

// multi-module, manual, custom mangler, stop before stripping:
let mut comp = Compilation::load(&root, &my_resolver, &options)?;   // the only I/O step
comp.validate()?;                                                    // "typecheck"
comp.mangle(&HashMangler);
let mut result = comp.assemble();                                    // consumes comp
my_transform(&mut result.syntax);                                    // raw AST access anytime
passes::lower(&mut result.syntax)?;
result.strip();
```

Consumers simplify: wesl-cli's compile collapses to one `.compile()` call site
(non-type-changing `set_resolver`); wesl-c stores a configured `Compiler` across FFI
calls; wesl-web drops its `use_sourcemap` special case and gains async.

---

# Migration & compatibility (target: wesl 0.5.0)

**Unchanged:** `Resolver`/`Mangler`/`SourceMap` traits + all impls (existing resolver
impls keep compiling byte-for-byte); `CompileResult` pub fields/Display/eval/exec;
`build_artifact` + `include_wesl!`; `PkgBuilder`/`wesl_pkg!`/wesl_toml; `Diagnostic`/`Error`;
`ModulePath`/`syntax` re-exports; `Preprocessor` (re-documented as advanced middleware).

**Deprecated (kept one release, removed 0.6):** `Wesl<R>` (thin shim; `pub type Wesl =
Compiler` is impossible — E0091 unused param — so the old struct stays verbatim behind
`#[deprecated(note = "use wesl::Compiler")]`); free `compile`/`compile_sourcemap`
(reimplemented over `Compilation`, so old paths run the new core).

**Breaking:** `Router` mounts gain `+ Send + Sync` bounds (so the default `Compiler` can
own one); `CompileOptions` gains `sourcemap: bool` (breaks exhaustive literals; in-repo
only — `..Default::default()` is the documented idiom); error text shows qualified names
for root decls.

**Docs:** README quickstart swaps `Wesl::new` → `Compiler::new` (same 3 lines) + new
"Custom transforms" / "Manual pipeline" / "Async loading" sections with doctests;
crates/wesl/CLAUDE.md + wesl-web/CLAUDE.md pipeline descriptions updated; CHANGELOG.

# Implementation plan (4 PRs, each leaves the tree green)

**PR 1 — Granular core (sync).** Create `compilation.rs` (`Compilation`,
`CompileResult` moves here + `keep` field + `strip()`), extract
`keep_idents`/`compile_pre_assembly`/`compile_post_assembly` logic from lib.rs into it;
create `passes.rs` (promote `condcomp::run`, `import::mangle_decls`,
`strip::strip_except`; wire `monomorphize`). Reimplement free `compile`/
`compile_sourcemap` on top (behavioral parity, esp. error decoration lib.rs:938-957).
Tests: byte-identical WGSL + error messages across wesl-test fixtures; a regression test
for the `assemble(self)`/use_count invariant; lazy-vs-eager assemble modes.
Files: `crates/wesl/src/{lib,import,compilation,passes,strip,condcomp}.rs`.

**PR 2 — Facade + hooks.** Create `compiler.rs` (`Compiler<R = Box<dyn Resolver + Send
+ Sync>>`, package/constant composition, `ManglerKind` handling) and `pass.rs`
(`ModulePass`/`ProgramPass`/`ModuleCtx`/`ProgramCtx`/`Stage`/`Passes`, blanket impls,
`Error::custom`); internal loader gains the module-pass chain (deleting the
`Preprocessor` wrap + `Box<dyn Resolver>` branch at lib.rs:844-851); `CompileOptions.sourcemap`;
`Router` bounds; deprecate `Wesl` + free fns; migrate wesl-cli, wesl-web, wesl-c,
examples, wesl-test, docs. Tests: pass ordering, pass-error context (module path + pass
name), worked examples end-to-end.
Files: `crates/wesl/src/{compiler,pass,resolve,lib}.rs`, `crates/wesl-{cli,web,c}/`, `examples/`.

**PR 3 — Sourcemap v2.** `WeakIdent` in wgsl-parse (additive); `DeclProvenance`/
`DeclLocation`/`BasicSourceMap` internals + `record_provenance` (end of load, pre-mangle)
+ `index_output_names` (at assemble); `output.rs` annotated writer +
`CompileResult::to_string_with_origins/annotated`; error.rs decl-span fallback.
Tests: strip-still-strips-with-sourcemap; `to_string_annotated(|_| None) == to_string()`;
golden annotated output with FileResolver (real paths) and VirtualResolver (degraded);
rename-pass + annotation (WeakIdent survives).
Files: `crates/wgsl-parse/src/syntax.rs`, `crates/wesl/src/{sourcemap,import,output,error}.rs`.

**PR 4 — Async.** `AsyncResolver`/`AsAsync`/forwarding impls (resolve.rs, ~90 LOC);
async-ify import.rs recursion (`Box::pin` placement per design, bodies otherwise
identical); `block_on_ready` + `async fn compile_impl` core; `Compilation::load_async`,
`Compiler::{load_async,compile_async}`; wesl-web `JsResolver` + `run_async`
(+`wasm-bindgen-futures`). Tests: sync path byte-identical (order + error attribution
are the regression surfaces); an actually-suspending fake async resolver end-to-end;
multi-module `AsAsync` exercise; perf sanity on a large fixture (<1% expected).
Files: `crates/wesl/src/{resolve,import,lib,compilation,compiler}.rs`, `crates/wesl-web/`.

PRs 3 and 4 are independent of each other; both build on 1–2.

**Risks, ranked:** (1) strip semantics via Arc use-counts — any stray graph/ident clone
surviving past `assemble` silently disables DCE → consuming signature + targeted tests;
(2) error-message parity while replacing `Preprocessor` wrapping and splitting
compile/compile_sourcemap → wesl-test snapshots errors, copy the exact diagnostic
construction; (3) async borrow/lifetime friction in boxed recursion → fallback: fn-level
box both recursive fns first, optimize after green; (4) closure-pass HRTB inference
(document required parameter annotations; struct impls always work).

# Verification

- `cargo test --workspace` + `cargo test -p wesl-test` (spec-tests, wesl-testsuite,
  bevy/wgpu fixtures) after each PR — output WGSL and error text must be byte-identical
  where behavior is meant to be unchanged.
- Doctests for every example in this plan (README + rustdoc).
- `cargo clippy --workspace --all-targets`, `cargo fmt --all`, feature matrix builds
  (`--no-default-features`, `--features eval,generics,package,naga-ext,serde`),
  `cargo doc` for the new API surface.
- End-to-end: run `examples/{buildrs_wesl,runtime_wesl,dependency-resolution}`;
  `wesl-cli compile` on samples/; wasm build of wesl-web (`run` unchanged, `run_async`
  with a JS Promise resolver).
