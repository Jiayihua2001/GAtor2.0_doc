# GAtor 2.0

<p align="center">
  <img src="assets/images/ga.png" alt="GAtor Logo" width="300">
</p>

**A First-Principles Genetic Algorithm for Molecular Crystal Structure Prediction**

<p align="center">
  <a href="https://github.com/Jiayihua2001/GAtor" class="md-button">
    GitHub Repository
  </a>
</p>

<p align="center">
  <img src="assets/images/innovation.png" alt="GAtor 1.0 to 2.0" width="600">
</p>

---

## What is GAtor?

GAtor is an open-source genetic algorithm (GA) for predicting the crystal structures of organic molecules. Given a molecular geometry, GAtor searches for the most stable packing arrangements in the solid state — a fundamental challenge in computational chemistry and materials science.

<p align="center">
  <img src="assets/images/GAtor2_target.png" alt="GAtor Target Systems" width="500">
</p>

### Key Features

<div class="grid cards" markdown>

- **Fast MLIP Screening** — UMA, MACE, AIMNet2 on GPU (~30 min for 300 generations)
- **DFT Backends** — FHI-aims (all-electron) and VASP for production-quality results
- **Cocrystals & Flexible Molecules** — Multi-component and conformer-aware operators
- **PXRD-Guided Search** — Combine energy with experimental diffraction data (VC-PWDF)
- **45+ Mutation Operators** — Hierarchical selection across translation, rotation, strain, block, and conformer categories
- **HPC-Ready** — Asynchronous parallel GA with `srun`/`mpirun`, scales to 16+ replicas

</div>

### How It Works

<p align="center">
  <img src="assets/images/GAtor_2.0.png" alt="GAtor 2.0 Workflow" width="450">
</p>

The GA loop evolves a pool of crystal structures through selection, crossover, mutation, and energy evaluation. Multiple replicas run in parallel on HPC clusters. The pool converges toward the global energy minimum.

---

## Get Started

**New to GAtor?** Follow this path:

[Install GAtor](getting-started/installation.md){ .md-button } [Run the Uracil Example](getting-started/quickstart.md){ .md-button .md-button--primary }

Or jump to a specific tutorial:

| Crystal Type | Tutorial | Time |
|---|---|---|
| Rigid molecule | [Uracil with UMA](tutorials/rigid-mlip.md) | ~30 min |
| Binary cocrystal | [BEDQAG (1:1)](tutorials/cocrystal.md) | ~1 hour |
| Flexible molecule | [UJIRIO with conformers](tutorials/flexible.md) | ~1 hour |
| PXRD-guided | [Uracil + experimental PXRD](tutorials/pxrd-assisted.md) | ~2 hours |

---

## Version History

### GAtor 2.0

Genetic Algorithm for Crystal Structure Prediction of Molecular Co-Crystals and Semi-Flexible Crystals.

**New in 2.0:**

- Co-crystal support with ASU partner swap operators
- Flexible molecule handling with conformer-aware mutations
- MLIP backends (UMA, MACE, AIMNet2) for GPU-accelerated screening
- PXRD-assisted fitness with VC-PWDF metric
- Asynchronous parallel architecture with multi-replica scaling

### GAtor 1.0

The original first-principles genetic algorithm for molecular crystal structure prediction.

> F. Curtis, X. Li, T. Rose, A. Vazquez-Mayagoitia, S. Bhattacharya, L. M. Ghiringhelli, and N. Marom,
> "GAtor: A First-Principles Genetic Algorithm for Molecular Crystal Structure Prediction",
> *J. Chem. Theory Comput.*, 2018, DOI: [10.1021/acs.jctc.7b01152](https://doi.org/10.1021/acs.jctc.7b01152)

---

## People

**GAtor 2.0 developer and maintainer:** Jiayi Huang

**GAtor 1.0 authors:** Farren Curtis, Xiayue Li, Timothy Rose, Alvaro Vazquez-Mayagoitia, Saswata Bhattacharya, Luca M. Ghiringhelli, and Noa Marom

**PI:** Noa Marom, Carnegie Mellon University

---

## Contact

For questions, bug reports, or feature requests:

- **Email:** Jiayi Huang — [jiayihua@andrew.cmu.edu](mailto:jiayihua@andrew.cmu.edu)
- **GitHub Issues:** [github.com/Jiayihua2001/GAtor/issues](https://github.com/Jiayihua2001/GAtor/issues)
