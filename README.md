# HyperBayes

**A next-generation Bayesian phylogenetics engine, in Rust and Python.**

> 🚧 **Under active development.** APIs are unstable and the project is pre-release.
> Phase 1 (BEAGLE FFI proof-of-concept) is complete; Phase 2 (I/O engine) is in progress.
> See [Roadmap](#roadmap).

**HyperBayes** is a next-generation Bayesian phylogenetic inference engine. It replaces the aging MrBayes C codebase with a memory-safe, highly parallel **Rust core** and a modern **Python interface**, using **libhmsbeagle** for hardware-accelerated likelihood evaluation.

Our long-term goal is to pioneer **Hyperbolic Variational Combinatorial Sequential Monte Carlo (H-VCSMC)** for high-dimensional tree inference, alongside a traditional Metropolis-coupled MCMC (MC³) engine.

---

## Current Status

| Component | Status |
| :--- | :--- |
| BEAGLE 3.1.2 FFI (Rust) | ✅ Complete |
| JC69 likelihood (Python ↔ Rust ↔ BEAGLE) | ✅ Complete |
| Python CLI (`hyperbayes run`) | ✅ Complete |
| Pattern compression | ✅ Complete |
| Newick parser | 🚧 In progress (Phase 2) |
| FASTA / NEXUS / PHYLIP parsers | 🚧 In progress (Phase 2) |
| MCMC engine (MC³, NNI, SPR) | 📋 Planned (Phase 3) |
| H-VCSMC + PyTorch | 📋 Planned (Phase 5) |

---

## Architecture

```
┌─────────────────────────┐
│   Python (typer CLI)    │  hyperbayes run -s data.fasta -t tree.nwk
└───────────┬─────────────┘
            │ pyo3
┌───────────▼─────────────┐
│   Rust (hyperbayes-py)  │  calculate_likelihood(seqs, newick)
└───────────┬─────────────┘
            │ safe FFI (bindgen)
┌───────────▼─────────────┐
│  Rust (hyperbayes-      │  BeagleInstance, BeagleOperation,
│        beagle)          │  JC69 eigen, transition matrices
└───────────┬─────────────┘
            │ C API
┌───────────▼─────────────┐
│  libhmsbeagle 3.1.2     │  CPU + SSE, double precision
└─────────────────────────┘
```

- **Rust** handles all tree data structures, pattern compression, and FFI. No Python in the hot path.
- **Python** handles CLI ergonomics, config files, and (in later phases) PyTorch-based VCSMC optimization.
- **BEAGLE 3.1.2** provides hardware-accelerated likelihoods. We deliberately target 3.1.2 (not 4.x) for ABI stability — it is the same version MrBayes 3.2.9 uses.

---

## Installation (Development)

HyperBayes is not yet on PyPI. To build from source:

### Prerequisites
- Rust 1.79+ (`rustup`)
- Python 3.9+
- `libhmsbeagle` 3.1.2 (via conda: `conda install -c bioconda beagle-lib=3.1.2`)
- A C linker on the system (`/usr/bin/cc`)

### Build the wheel
> ⚠️ Rust must be built with the **system** compiler, not conda's. The Python wheel can be installed inside conda.

```bash
conda activate py_latest_devel
export CC=/usr/bin/cc
export CXX=/usr/bin/c++
export LD=/usr/bin/ld
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib
maturin build --release
pip install target/wheels/hyperbayes-*.whl --no-deps
```

### Run

```bash
hyperbayes version
hyperbayes run -s benchmark/test.fasta -b 0.02
```

Output:
```
Loaded 5 taxa from test.fasta
Computing likelihood with branch length 0.02...
Log-Likelihood: -46.368610
```

---

## Roadmap

| Phase | Goal | Status |
| :--- | :--- | :--- |
| **0** | Environment setup (MrBayes 3.2.9 + BEAGLE 3.1.2) | ✅ Complete |
| **1** | FFI proof-of-concept (Rust ↔ BEAGLE ↔ Python) | ✅ Complete |
| **2** | I/O engine (Newick, FASTA, NEXUS, PHYLIP) | 🚧 In progress |
| **3** | MCMC engine (JC/HKY/GTR, MC³, NNI/SPR) | 📋 Planned |
| **4** | Complex models & partitions | 📋 Planned |
| **5** | H-VCSMC + PyTorch | 📋 Planned |
| **6** | Release (PyPI, Bioconda, Crates.io) | 📋 Planned |

---

## Design Decisions

### Why BEAGLE 3.1.2 and not 4.x?
BEAGLE 4.0.1 introduced an ABI change to `beagleCreateInstance` (return-value vs. output-pointer) and silent `handle=-1` failures that make it unsuitable for production. BEAGLE 3.1.2 is:
- The same version shipped with MrBayes 3.2.9 (proven stable).
- Simpler C API with predictable behavior.
- Fully sufficient for Phase 1–3; GPU/CUDA support is deferred to Phase 5.

### Why a Rust core?
- **Memory safety.** No more segfaults from buffer overruns in MCMC moves.
- **Parallelism.** `rayon`-based MCMC chains will scale across cores.
- **Python compatibility.** `pyo3` + `maturin` gives us a first-class Python package without re-writing the hot path in Cython.

### Why PyTorch for VCSMC and not BEAGLE gradients?
Per the BEAGLE 4.0.1 API, no gradient functions are exposed. VCSMC will use PyTorch's `autograd` for tree-traversal gradients, calling BEAGLE for the likelihood scalars (via `pyo3-dlpack` zero-copy tensors).

---

## Reference Benchmarks

TBA, pending testing.

---

## License

GNU AGPL-3.0-or-later. See [LICENSE](LICENSE).

---

## Contributing

The project is in early development. If you are interested in contributing (Rust FFI, phylogenetics, or PyTorch), open an issue or reach out. The roadmap and design decisions are documented in `docs/`.

---

## References

- **MrBayes 3.2** [(Ronquist et al., 2012)](https://doi.org/10.1093/sysbio/sys029) — for the reference implementation and baseline.
- **BEAGLE** [(Suchard et al., 2011)](https://doi.org/10.1093/sysbio/syr100) — for the hardware-accelerated likelihood core.
- **ModelFinder** [(Kalyaanamoorthy et al., 2017)](https://doi.org/10.1038/nmeth.4285) and **IQ-TREE 3** [(Wong et al., 2026)](https://doi.org/10.1093/molbev/msag117) — for the model definitions and test comparisons.
- **VCSMC Algorithm** [(Chen et al., 2025)](https://proceedings.mlr.press/v258/chen25f.html) — for the VCSMC methodology that Phase 5 will implement.
