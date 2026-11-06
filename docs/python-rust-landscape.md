# Python-to-Rust transpilation and Python execution from Rust

Research date: **2026-11-06**.

## Findings

**Existing transpilers already cover parts of this project's goal. Evaluate Depyler and py2many before building a new translation engine.** Depyler is particularly relevant because it exposes Rust crates; py2many provides another implementation and a useful Python/Rust example corpus. Neither has been validated against this project's requirements.

Running Python inside a Rust program is a separate option: PyO3 embeds an existing interpreter, while RustPython implements an interpreter in Rust. Those approaches preserve a Python execution layer rather than producing idiomatic Rust source.

At the start of this research, this repository contained only its purpose in `README.md`. Everything recommended below is **proposed**, with no parser, transpiler, runtime integration, or dependencies added here.

## Scope and evidence

Discovery used web search, followed by official repositories, source files, documentation, and crates.io metadata. A direct Google search-page request was attempted but the research tool could not access it; the available web search was used instead. Example discovery queries:

- `Python Rust transpiler py2many mypy`
- `Python to Rust transpiler crates depyler`
- `Rust run Python PyO3 RustPython rust-cpython pyembed`
- `RustPython Parser ruff_python_parser mypyc design typed Python`

No candidate was installed, built, executed, benchmarked, or audited for semantic equivalence. “Supported” in upstream documentation is an upstream claim, not a guarantee established here. Crate publication and source inspection establish that artifacts exist; they do not establish that the artifacts work together. Repository default branches can differ from published releases.

## Existing transpilers

| Project | Implementation / distribution | Relationship to our goal | Suggested role |
| --- | --- | --- | --- |
| [Depyler](https://github.com/paiml/depyler) | Rust workspace; [`depyler`](https://crates.io/crates/depyler) and [`depyler-core`](https://crates.io/crates/depyler-core) crates | Annotated Python to Rust | First reuse candidate |
| [py2many](https://github.com/py2many/py2many) | Python tool with a Rust backend | Python to Rust and other languages | Alternative engine; reference corpus and lowering rules |
| [konchunas/pyrs](https://github.com/konchunas/pyrs) | Python syntax converter | Earlier Python-to-Rust project with the same name | Historical lessons; directs users to py2many |
| [Rextio](https://github.com/rextio/rextio) | Python build tool generating Rust/PyO3 | Selected typed functions compiled ahead of time, with Python fallback | Reference for conservative acceptance and hybrid boundaries |

### Depyler: inspect first for reusable Rust components

The [README](https://github.com/paiml/depyler/blob/cd045f371c7ffe5b4d937bd45a7bfb016775a56f/README.md) describes annotation-directed translation through a Python AST, an intermediate representation, type analysis, and Rust generation. It reports unsupported dynamic operations such as `eval`, `exec`, reflection, and monkey patching. Its compile-rate and verification claims require independent testing; compilation alone does not prove matching behavior.

Actual source is present: [`depyler-core/src/lib.rs`](https://github.com/paiml/depyler/blob/cd045f371c7ffe5b4d937bd45a7bfb016775a56f/crates/depyler-core/src/lib.rs) exposes `DepylerPipeline`, documents parsing and ownership analysis, and re-exports analysis components. Its [manifest](https://github.com/paiml/depyler/blob/cd045f371c7ffe5b4d937bd45a7bfb016775a56f/crates/depyler-core/Cargo.toml) depends on `rustpython-parser`, `syn`, `quote`, and workspace crates.

**Evaluation:** test both the published CLI and library API against the same small corpus. Determine whether using the engine, contributing missing behaviors upstream, or adapting selected components is cheaper than implementing them independently. Do not assume that its annotation support consumes Mypy's inferred types or guarantees Mypy compatibility.

### py2many: another engine and a readable reference implementation

The [README](https://github.com/py2many/py2many/blob/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/README.md) identifies Rust as its primary target and traces its history to the older pyrs. It is a Python tool, rather than a Rust library dependency.

Its [`pyrs/transpiler.py`](https://github.com/py2many/py2many/blob/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/pyrs/transpiler.py) contains AST visitors, Rust-specific rewrites, and structured Rust rendering. [`pyrs/inference.py`](https://github.com/py2many/py2many/blob/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/pyrs/inference.py) contains concrete type mappings, including Python `int` to `i32` and collection mappings. [`pyrs/plugins.py`](https://github.com/py2many/py2many/blob/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/pyrs/plugins.py) is a useful entry point for library-call translation rules. Compare [`tests/cases`](https://github.com/py2many/py2many/tree/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/tests/cases) with [`tests/expected`](https://github.com/py2many/py2many/tree/937d999ad29161b4e0a0f6d88ace16cfd42b05e2/tests/expected).

**Evaluation:** inspect generated Rust for ownership choices, runtime dependencies, unsupported constructs, and behavioral changes. In particular, fixed-width integer mapping needs explicit overflow tests. Avoid treating an attractive example or feature checklist as proof of general Python compatibility.

### Historical pyrs: naming collision and useful warnings

[`konchunas/pyrs`](https://github.com/konchunas/pyrs) is a separate repository, not an earlier version of this checkout. It is archived. Its README explicitly says it is unmaintained and redirects readers to py2many. It warns that output may fail to compile, allocate unnecessarily, or silently skip statements; unknown types can become placeholders.

**Lesson for this project:** reject unsupported syntax with source locations instead of silently emitting incomplete Rust. Distinguish generated migration scaffolding from compilable, behavior-preserving output. Explain the naming distinction if this project becomes public-facing beyond its current README.

### Rextio: study the supported-subset boundary

The [README](https://github.com/rextio/rextio/blob/694845d7673b3bd5248ba509b91d214b44a193b0/README.md) describes an alpha tool that lowers accepted typed functions to Rust/PyO3 and keeps other functions on Python fallback. It explicitly restricts native/fallback call graphs and mutable-container crossings. It distinguishes direct native execution from shims that call Python.

Concrete entry points include its [analyzer](https://github.com/rextio/rextio/tree/694845d7673b3bd5248ba509b91d214b44a193b0/src/rextio/analyzer) and [Rust generation code](https://github.com/rextio/rextio/tree/694845d7673b3bd5248ba509b91d214b44a193b0/src/rextio/codegen/rust), including checked arithmetic and type mapping.

**Lesson:** eligibility diagnostics and fallback policy deserve explicit design. Hybrid acceleration is an alternative product direction; it should not be silently added to a standalone source-to-source transpiler.

## Ways to run Python from Rust

| Tool | How it runs Python | Useful application here | Boundary |
| --- | --- | --- | --- |
| [PyO3](https://github.com/PyO3/pyo3), crate [`pyo3`](https://crates.io/crates/pyo3) | Calls an interpreter in the Rust process | Invoke Python libraries; optional generated-extension bindings | Interpreter and linking requirements remain |
| [RustPython](https://github.com/RustPython/RustPython), crate [`rustpython-vm`](https://crates.io/crates/rustpython-vm) | Executes Python using a VM implemented in Rust | Embedded scripting; implementation reference | Python/package compatibility must be checked |
| [`inline-python`](https://github.com/m-ou-se/inline-python) | Inline Python macro backed by PyO3 | Convenient small interop experiments | Python remains interpreted; Rust tokenization limits syntax |
| [`pyembed`](https://docs.rs/pyembed/latest/pyembed/) / [PyOxidizer](https://github.com/indygreg/PyOxidizer) | Configures and packages an embedded Python interpreter | Study runtime distribution if hybrid execution is chosen | Packaging is not transpilation; older published crate |
| [`cpython`](https://github.com/dgrunwald/rust-cpython) | CPython bindings | Historical comparison | Maintainers direct new users to PyO3 |
| [`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) | Starts an external Python process | Simple reference runner for differential tests | Requires a chosen Python executable and data exchange |

### PyO3

The [official guide](https://pyo3.rs/) covers embedding Python in Rust and exposing Rust modules to Python. The [`Python` API](https://docs.rs/pyo3/latest/pyo3/marker/struct.Python.html) provides execution/evaluation and value conversion. Embedding requires appropriate interpreter configuration and linking; exporting an extension is a different build mode.

**Recommendation:** use PyO3 if in-process Python integration becomes a requirement. It can also expose generated Rust to existing Python tests. It does not automatically translate Python functions, remove the runtime, or make Python code faster merely by embedding it. Select documentation for the pinned crate version when experimenting.

### RustPython

RustPython's [README](https://github.com/RustPython/RustPython#embedding-rustpython-into-your-rust-applications) points to embedding examples and warns that the interpreter remains in development. Its [architecture document](https://github.com/RustPython/RustPython/blob/main/architecture/architecture.md) is useful for understanding parsing, compilation, and VM separation.

**Recommendation:** consider it when a Rust implementation of Python execution is itself desirable. Verify required standard-library behavior and native-extension compatibility instead of assuming it substitutes for CPython. It produces neither a native Rust translation of the application nor automatically idiomatic Rust source.

### Smaller and older embedding options

[`inline-python`](https://github.com/m-ou-se/inline-python) offers a `python!` macro and shared contexts. Its README documents tokenizer restrictions, including string/comment syntax. This is convenience around interpreter execution.

[`pyembed`](https://docs.rs/pyembed/latest/pyembed/) manages interpreter configuration and lifecycle using PyO3 underneath. PyOxidizer focuses on distribution, including embedded resources. Investigate current toolchain compatibility before adopting it.

[`rust-cpython`](https://github.com/dgrunwald/rust-cpython) explicitly says it is no longer actively maintained and recommends PyO3. It is not the preferred foundation for a new integration.

For initial correctness comparisons, an external CPython process is the simplest proposed reference runner: it avoids an embedding dependency and keeps the source interpreter separate from generated code. Process execution alone is not a sandbox.

## Frontends and Mypy-related inspiration

### Reuse a Python parser, but distinguish parsing from type checking

- [`rustpython-parser`](https://crates.io/crates/rustpython-parser) is a published parser library. Its [repository README](https://github.com/RustPython/Parser/blob/287eb807f92a8470448c7d63b51e36c176297f1e/README.md) says the parser has been superseded by Ruff's parser, while keeping packaged libraries available. It remains relevant to existing consumers such as Depyler.
- Ruff's [`ruff_python_parser`](https://github.com/astral-sh/ruff/tree/main/crates/ruff_python_parser) is another candidate. RustPython's [workspace manifest](https://github.com/RustPython/RustPython/blob/main/Cargo.toml) references republished `rustpython-ruff_*` packages; do not confuse those package names with the older `rustpython-parser` API. Verify publication, AST API, syntax coverage, and version compatibility before choosing.

A parser supplies syntax trees. It does not supply Mypy's name resolution, inferred types, narrowing, or plugin behavior. “Mypy-annotated Python” needs a definition: Python with accepted annotations, or specifically the results of running Mypy?

### mypyc is a close design reference, with a different output target

[mypyc](https://mypyc.readthedocs.io/en/latest/introduction.html) compiles typed Python modules to C extensions using Mypy for type checking and inference. It uses a restricted, gradually typed Python variant and can interoperate with ordinary Python modules. It does not emit Rust.

**Lesson:** study how an annotation-driven compiler documents its supported language and runtime boundaries. If actual Mypy results are required, investigate a versioned adapter rather than assuming annotations alone contain all needed type information. No stable Mypy-to-Rust interface was established by this research.

## Design choices to settle before implementation

These are recommendations derived from the research, not adopted architecture.

| Question | Why it matters | Proposed initial position |
| --- | --- | --- |
| Standalone Rust or Python extension? | Determines runtime dependencies and callable interfaces | Evaluate standalone output first; assess bindings separately |
| What qualifies as idiomatic output? | Compiling output can still be difficult to maintain | Review ownership, signatures, clones, error handling, and readability |
| What does Python `int` mean? | Python integers are arbitrary precision; Rust primitives are bounded | Preserve semantics or explicitly constrain/reject ranges; never silently narrow |
| Can two names alias one list? | Moving or cloning a `Vec` can change observable mutation | Reject unsupported alias patterns until their semantics are modeled |
| How are imports handled? | A Python package name rarely maps directly to a Rust crate | Maintain explicit mappings; diagnose unknown imports |
| What happens to unsupported code? | Silent omission can produce plausible but incorrect programs | Emit actionable errors with source spans |
| Are fallback calls allowed? | They retain a Python runtime and complicate data ownership | Make fallback an explicit mode if later justified |

Additional semantic tests should cover negative floor division and modulo, negative indexing, Unicode operations, dictionary iteration order, truthiness, exception paths, and evaluation order. Type annotations do not resolve these differences by themselves.

## Proposed evaluation sequence

1. **Define a small accepted subset:** annotated functions, scalar arithmetic, branching, loops, and a few collection operations. Record exclusions and numeric semantics.
2. **Evaluate existing tools:** run published Depyler and py2many against identical fixtures before selecting a new implementation path. Record exact versions and commands.
3. **Measure three different outcomes:** translation succeeds, generated Rust compiles, and execution agrees with CPython. Report failures separately; add manual readability review.
4. **Exercise boundaries:** include unsupported syntax and semantic edge cases. Require explicit rejection rather than incomplete output or accidental success.
5. **Choose reuse level:** use an existing CLI, depend on a verified library API, contribute upstream, or implement only the demonstrated missing behavior.

Possible future experiments, **not run during this research**:

```sh
# In a disposable evaluation environment after choosing versions:
depyler transpile fixtures/example.py
py2many --rust fixtures/example.py
```

The fixture directory and commands above are illustrative; this repository does not yet contain fixtures or either tool. Installing and running candidates is separate work.

## Publication and maintenance snapshot

Versions below are crates.io's latest stable versions observed on the research date. They are discovery data, not selected dependencies.

| Crate | Stable version | Registry update date |
| --- | --- | --- |
| [`depyler`](https://crates.io/crates/depyler) | 4.1.1 | 2026-02-15 |
| [`depyler-core`](https://crates.io/crates/depyler-core) | 4.1.1 | 2026-02-15 |
| [`pyo3`](https://crates.io/crates/pyo3) | 0.30.1 | 2026-10-14 |
| [`rustpython-vm`](https://crates.io/crates/rustpython-vm) | 0.6.0 | 2026-10-02 |
| [`rustpython-parser`](https://crates.io/crates/rustpython-parser) | 0.4.0 | 2024-08-06 |
| [`inline-python`](https://crates.io/crates/inline-python) | 0.14.4 | 2025-06-27 |
| [`pyembed`](https://crates.io/crates/pyembed) | 0.24.0 | 2022-12-29 |
| [`cpython`](https://crates.io/crates/cpython) | 0.7.2 | 2024-07-02 |

Repository metadata was read through the [GitHub REST API](https://docs.github.com/en/rest/repos/repos#get-a-repository); registry metadata through the corresponding `https://crates.io/api/v1/crates/{name}` endpoints. GitHub reported recent pushes for Depyler (2026-11-05) and py2many (2026-11-03), and `archived: true` for konchunas/pyrs. PyOxidizer was not archived, but its last reported push was 2024-12-24. Push timestamps alone do not establish maintenance quality or release compatibility.

Source snapshots used for detailed inspection:

| Repository | Inspected commit |
| --- | --- |
| py2many/py2many | [`937d999`](https://github.com/py2many/py2many/tree/937d999ad29161b4e0a0f6d88ace16cfd42b05e2) |
| paiml/depyler | [`cd045f3`](https://github.com/paiml/depyler/tree/cd045f371c7ffe5b4d937bd45a7bfb016775a56f) |
| rextio/rextio | [`694845d`](https://github.com/rextio/rextio/tree/694845d7673b3bd5248ba509b91d214b44a193b0) |
| RustPython/Parser | [`287eb80`](https://github.com/RustPython/Parser/tree/287eb807f92a8470448c7d63b51e36c176297f1e) |

GitHub's detected licenses for these four repositories and konchunas/pyrs were MIT. It reported BSD-2-Clause for inline-python and MPL-2.0 for the PyOxidizer repository. Check each actual component's license and preserve required notices before copying implementation code; repository-level metadata is not a component license audit. This research copies no implementation code.
