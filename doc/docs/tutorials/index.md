# Tutorials

Step-by-step guides for common use cases.

| Tutorial | Description | Difficulty |
|----------|-------------|------------|
| [Rigid Molecule with MLIP](rigid-mlip.md) | Predict crystal structures using UMA or MACE on GPUs | Beginner |
| [Cocrystal Prediction](cocrystal.md) | Multi-component crystal structure prediction | Intermediate |
| [Flexible Molecule](flexible.md) | Handle conformational flexibility during CSP | Intermediate |
| [PXRD-Assisted Search](pxrd-assisted.md) | Use experimental PXRD data to guide the search | Advanced |

---

## Before You Begin

All tutorials assume:

1. GAtor is [installed](../getting-started/installation.md) and the `gator` Conda environment is activated
2. You have prepared `structures.json` (see [Quick Start — Preparation Checklist](../getting-started/quickstart.md#preparation-checklist))
3. You have access to a GPU node (for MLIP runs) or CPU nodes (for DFT runs)

## Example Files

Complete, ready-to-use configuration files are provided in the GAtor repository under `example/`:

```
example/
├── prepare_structures.py          # Helper: convert CIF/POSCAR to structures.json
├── rigid/
│   └── MLIP/
│       ├── ui_gpu_srun.conf       # Rigid molecule + UMA (srun)
│       ├── ui_gpu_mpirun.conf     # Rigid molecule + UMA (mpirun)
│       └── submit_gpu.sh          # SLURM script
├── flexible/
│   ├── ui.conf                    # Flexible molecule + MACE
│   ├── submit_gpu.sh              # SLURM script
│   └── prepare_conformers.py      # Helper: generate conformer pool with RDKit
└── cocrystal/
    └── MLIP/
        ├── ui.conf                # Cocrystal + UMA
        └── submit_gpu.sh          # SLURM script
```
