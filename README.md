# HyperBayes

**A next-generation Bayesian phylogenetics engine, in Rust and Python.**

> 🚧 **Under active development.** APIs are unstable and the project is pre-release.
> Phase 1 (BEAGLE FFI proof-of-concept) and Phase 2 (I/O engine) are complete.
> See [Roadmap](#roadmap).

**HyperBayes** is a next-generation Bayesian phylogenetic inference engine. It replaces the aging MrBayes C codebase with a memory-safe, highly parallel **Rust core** and a modern **Python interface**, using **libhmsbeagle** for hardware-accelerated likelihood evaluation.

Our long-term goal is to pioneer **Hyperbolic Variational Combinatorial Sequential Monte Carlo (H-VCSMC)** for high-dimensional tree inference, alongside a traditional Metropolis-coupled MCMC (MC³) engine.

---

## Current Status

| Component | Status |
| :--- | :--- |
| BEAGLE 3.1.2 FFI (Rust) | ✅ Complete |
| NEXUS alignment parser | ✅ Complete |
| Newick parser (+ root-polytomy ladderization) | ✅ Complete |
| Dynamic BEAGLE operation plan from parsed tree | ✅ Complete |
| JC69 likelihood (Python ↔ Rust ↔ BEAGLE) | ✅ Complete — reproduces `-64.738781` |
| Python CLI (`hyperbayes run -s … -t …`) | ✅ Complete |
| Pattern compression (8 patterns on `tiny.nex`) | ✅ Complete |
| FASTA parser | ✅ Complete |
| PHYLIP / CLUSTAL / MSF parsers | 📋 Planned (Phase 3+) |
| `nexus::read_trees` (N trees from a `.t` file) | 📋 Planned (Phase 3 first step) |
| MCMC engine (MC³, NNI, SPR, estimated Pi) | 📋 Planned (Phase 3) |
| Complex models & partitions | 📋 Planned (Phase 4) |
| H-VCSMC + PyTorch | 📋 Planned (Phase 5) |

---

## Architecture

```
┌─────────────────────────┐
│   Python (typer CLI)    │  hyperbayes run -s tiny.nex -t tiny.newick
└───────────┬─────────────┘
            │ pyo3
┌───────────▼─────────────┐
│  Rust (hyperbayes-py)   │  calculate_likelihood(alignment_path, tree_path)
└───────────┬─────────────┘
            │
┌───────────▼─────────────────────────────────────┐
│  Rust (hyperbayes-core)                         │
│  parse_nexus → parse_newick → plan_operations   │
└───────────┬─────────────────────────────────────┘
            │
┌───────────▼─────────────────────────┐
│  Rust (hyperbayes-beagle)           │  BeagleInstance, BeagleOperation
└───────────┬─────────────────────────┘
            │ safe FFI (bindgen)
┌───────────▼─────────────────────────┐
│  libhmsbeagle 3.1.2                 │  CPU + SSE, double precision
└─────────────────────────────────────┘
```

- **Rust** handles all tree data structures, alignment parsing, pattern compression, operation planning, and FFI. No Python in the hot path.
- **Python** handles CLI ergonomics and (in later phases) ArviZ diagnostics and PyTorch-based VCSMC optimization.
- **BEAGLE 3.1.2** provides hardware-accelerated likelihoods. We deliberately target 3.1.2 (not 4.x) for ABI stability — it is the same version MrBayes 3.2.9 uses.

---

## Installation (Development)

HyperBayes is not yet on PyPI. To build from source:

### Prerequisites

- Rust ≥ 1.87 (`rustup`) — `.is_multiple_of` and other recent APIs are used.
- Python ≥ 3.9 (abi3 wheel; tested on 3.14.6 in conda).
- `libhmsbeagle` 3.1.2.
  - System: `apt install libhmsbeagle-dev` (Ubuntu 24.04 ships 3.1.2), or
  - Conda: `conda install -c bioconda beagle-lib=3.1.2`.
- A system C linker: `/usr/bin/cc`.

### Split-environment build (SDP §3.1)

Rust must be built with rustup's `cargo` and the **system** linker, not
conda's. Python runs inside conda. If `conda activate` is used, the env
must arrange for `~/.cargo/bin` to precede `$CONDA_PREFIX/bin` on `PATH`,
and cargo must be told to use `/usr/bin/cc`.

The repo carries `.cargo/config.toml` with `linker = "/usr/bin/cc"` so
`cargo` from any clone links correctly. Inside the `py_latest_devel`
conda env, `$CONDA_PREFIX/etc/conda/activate.d/rust.sh` sets the same
linker and prepends `~/.cargo/bin` to `PATH`.

Verify:

```bash
which cargo && cargo --version                # must be ~/.cargo/bin, ≥ 1.87
echo "$CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER"  # /usr/bin/cc
pkg-config --modversion hmsbeagle-1           # 3.1.2
```

### Build and install

```bash
conda activate py_latest_devel
maturin develop --release -m crates/hyperbayes-py/Cargo.toml
# or, for a self-contained wheel:
maturin build --release
pip install target/wheels/hyperbayes-*.whl --no-deps --force-reinstall
```

`maturin build` bundles `libhmsbeagle.so.1.3.2` into `hyperbayes.libs/`
inside the wheel, so the installed package is self-contained at runtime.

### Run

In current state,

```bash
hyperbayes version
hyperbayes run -s benchmark/tiny.nex -t benchmark/tiny.newick
```

Output:

```
hyperbayes 0.1.0  (BEAGLE 3.1.2)
Log-Likelihood: -64.738781
```

---

## Reference Benchmarks

TBA, pending testing

---

## Roadmap

| Phase | Goal | Status |
| :--- | :--- | :--- |
| **0** | Environment setup (MrBayes 3.2.9 + BEAGLE 3.1.2) | ✅ Complete |
| **1** | FFI proof-of-concept (Rust ↔ BEAGLE ↔ Python) | ✅ Complete |
| **2** | I/O engine (NEXUS + Newick, dynamic operation plan) | ✅ Complete |
| **3** | MCMC engine (JC/HKY/GTR, MC³, NNI/SPR, estimated Pi) | 🚧 Next |
| **4** | Complex models & partitions (PHYLIP/CLUSTAL/MSF) | 📋 Planned |
| **5** | H-VCSMC + PyTorch | 📋 Planned |
| **6** | Release (PyPI, Bioconda, Crates.io) | 📋 Planned |

Phase 3 first gate: reproduce `run1.p` gen-0 `lnLike = -65.763371` on the
starting tree of `tiny.nex.run1.t` under JC69 / `Pi=0.25` to 1e-6. That
proves the FFI can be called from an MCMC loop without state corruption.

---

## Design Decisions

### Why BEAGLE 3.1.2 and not 4.x?

BEAGLE 4.0.1 introduced an ABI change to `beagleCreateInstance`
(return-value vs. output-pointer) and silent `handle=-1` failures that
make it unsuitable for production. Our `hyperbayes-beagle/build.rs`
rejects 4.x at build time via
`pkg_config::range_version("3.1.0".."3.2.0")`. BEAGLE 3.1.2 is:

- The same version shipped with MrBayes 3.2.9 (proven stable).
- Simpler C API with predictable behavior.
- Fully sufficient for Phases 1–4; GPU/CUDA support is deferred to Phase 5.

### Why a Rust core?

- **Memory safety.** No more segfaults from buffer overruns in MCMC moves.
- **Parallelism.** `rayon`-based MCMC chains will scale across cores.
- **Python compatibility.** `pyo3` + `maturin` gives us a first-class Python package
  without re-writing the hot path in Cython.

### Why is the BEAGLE operation plan derived from the tree?

The Phase-1 hardcoded `BeagleOperation` array only worked for one
topology. `hyperbayes-core::beagle_ops::plan_operations` walks the parsed
tree in post-order and emits one op per internal node. Two invariants
matter:

- **Leaf partial buffers occupy `0..n_taxa`.** Internal partial buffers
  start at `n_taxa`. If they start at `0`, a freed tip buffer can be
  reused as a destination whose left child is that same tip, producing a
  `dest == child` op that corrupts the SSE-double `beagleUpdatePartials`
  in place (observed as `SIGSEGV`).
- **`dest != c1 && dest != c2`** for every op. This is asserted in the
  planner.

### Why does the Newick parser ladderize the root?

MrBayes emits unrooted trees with a trifurcating root, e.g.
`(2,(5,(4,3)),1)`. BEAGLE (and any binary operation plan) requires a
binary root. `Tree::resolve_root_polytomy` inserts a 0.0-length basal
edge, turning the trifurcation into a binary ladder. Under JC69/GTR the
0.0-length edge is a pure pass-through, so the likelihood is unchanged.

### Why PyTorch for VCSMC and not BEAGLE gradients?

No gradient functions are exposed by the BEAGLE C API. VCSMC will use
PyTorch's `autograd` for tree-traversal gradients, calling BEAGLE for
the likelihood scalars (via `pyo3-dlpack` zero-copy tensors).

---

## Testing

```bash
cargo test --workspace                # core + beagle + vi unit tests
cargo clippy --workspace -- -D warnings
pytest tests/ -v                      # smoke: CLI + FFI baseline
```

Expected:

- `cargo test --workspace` — 9 + 1 passing.
- `cargo clippy --workspace -- -D warnings` — clean.
- `pytest tests/ -v` — 2 passing. `test_ffi_proto_runs_and_prints_likelihood`
  asserts `abs(lik - (-64.738781)) < 1e-3`.

---

## License

GNU AGPL-3.0-or-later. See [LICENSE](LICENSE).

---

## Contributing

The project is in early development. If you are interested in
contributing (Rust FFI, phylogenetics, or PyTorch), open an issue or
reach out. The roadmap and design decisions are documented in this
README and in the SDP (`Konsep HyperBayes 1.3`).

---

## References

- **MrBayes 3.2** [(Ronquist et al., 2012)](https://doi.org/10.1093/sysbio/sys029) —
  reference implementation and baseline.
- **BEAGLE** [(Suchard et al., 2011)](https://doi.org/10.1093/sysbio/syr100) —
  hardware-accelerated likelihood core.
- **ModelFinder** [(Kalyaanamoorthy et al., 2017)](https://doi.org/10.1038/nmeth.4285) —
  model selection.
- **IQ-TREE 3** [(Wong et al., 2026)](https://doi.org/10.1093/molbev/msag117) —
  model definitions and test comparisons.
- **VCSMC** [(Chen et al., 2025)](https://proceedings.mlr.press/v258/chen25f.html) —
  the methodology Phase 5 will implement.

