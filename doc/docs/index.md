# GAtor 2.0

<p align="center">
  <img src="assets/images/ga.png" alt="GAtor Logo" width="300">
</p>

**A First-Principles Genetic Algorithm for Molecular Crystal Structure Prediction**

[Get Started](getting-started/installation.md){ .md-button .md-button--primary }  [GitHub Repository](https://github.com/Jiayihua2001/GAtor){ .md-button }

---

## What is GAtor?

Predicting how a molecule packs into a crystal is one of the hardest open problems in computational chemistry — crystal packing determines properties such as solubility, bioavailability, and stability. GAtor is an open-source genetic algorithm (GA) that searches the potential energy landscape of molecular crystals to find the most stable packing arrangements.

!!! note "Prerequisites"
    GAtor assumes familiarity with the Linux command line and access to an HPC cluster. MLIP-only runs have no license requirements; DFT-level runs require a licensed copy of FHI-aims or VASP.

<p align="center">
  <img src="assets/images/GAtor2_target.png" alt="GAtor Target Systems" width="500">
</p>

---

## Get Started

**New to GAtor?** Start here:

1. Complete [Installation](getting-started/installation.md)
2. Run [Tutorial 1: Setup & Preparation](tutorials/setup.md)
3. Pick the tutorial that matches your system:

| Crystal Type | Tutorial | Time |
|---|---|---|
| Rigid molecule | [Uracil with UMA](tutorials/rigid-mlip.md) | ~30 min |
| Binary cocrystal | [BEDQAG (1:1)](tutorials/cocrystal.md) | ~1 hour |
| Flexible molecule | [UJIRIO with conformers](tutorials/flexible.md) | ~1 hour |
| PXRD-guided | [Uracil + experimental PXRD](tutorials/pxrd-assisted.md) | ~2 hours |

---

## Key Features

<div class="grid cards" markdown>

- **Fast MLIP Screening** — UMA, MACE, AIMNet2 on GPU (~30 min for 300 generations)
- **DFT Backends** — FHI-aims (all-electron) and VASP for production-quality results
- **Cocrystals & Flexible Molecules** — Multi-component and conformer-aware operators
- **PXRD-Guided Search** — Combine energy with experimental diffraction data (VC-PWDF)
- **45+ Mutation Operators** — Hierarchical selection across translation, rotation, strain, block, and conformer categories
- **HPC-Ready** — Asynchronous parallel GA with `srun`/`mpirun`, scales to 16+ replicas

</div>

---

## How It Works

<p align="center">
  <img src="assets/images/GAtor_2.0.png" alt="GAtor 2.0 Workflow" width="450">
</p>

The GA loop evolves a pool of crystal structures through selection, crossover, mutation, and energy evaluation. Multiple replicas run in parallel on HPC clusters. The pool converges toward the global energy minimum.

**Learn more:** [User Guide](user-guide/index.md) | [Modules](modules/index.md) | [Configuration Reference](reference/config-reference.md)

---

## Versions

### GAtor 2.0

Genetic Algorithm for Crystal Structure Prediction of Molecular Co-Crystals and Semi-Flexible Crystals — Jiayi Huang and Noa Marom.

<p align="center">
  <img src="assets/images/innovation.png" alt="GAtor 1.0 to 2.0" width="600">
</p>

**New in 2.0:** co-crystal support, flexible molecule handling with conformer-aware mutations, MLIP backends (UMA, MACE, AIMNet2) for GPU-accelerated screening, PXRD-assisted fitness (VC-PWDF), and asynchronous parallel architecture with multi-replica scaling.

### GAtor 1.0

The original first-principles genetic algorithm for molecular crystal structure prediction — Farren Curtis, Xiayue Li, Timothy Rose, Alvaro Vazquez-Mayagoitia, Saswata Bhattacharya, Luca M. Ghiringhelli, and Noa Marom.

> *J. Chem. Theory Comput.*, 2018, 14(4), 2246-2264. DOI: [10.1021/acs.jctc.7b01152](https://doi.org/10.1021/acs.jctc.7b01152)

---

## Contact & Credits

**GAtor 2.0 developer and maintainer:** Jiayi Huang ([jiayihua@andrew.cmu.edu](mailto:jiayihua@andrew.cmu.edu))

**PI:** Noa Marom, Carnegie Mellon University

**GAtor 1.0 authors:** Farren Curtis, Xiayue Li, Timothy Rose, Alvaro Vazquez-Mayagoitia, Saswata Bhattacharya, Luca M. Ghiringhelli, and Noa Marom

For questions, bug reports, or feature requests: [GitHub Issues](https://github.com/Jiayihua2001/GAtor/issues)
