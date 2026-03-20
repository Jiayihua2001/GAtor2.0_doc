# Tutorial 4: Flexible Molecule

Predict the crystal structure of **UJIRIO** — a molecule with rotatable bonds — using conformer-aware mutations and torsion angle blending.

!!! info "Prerequisites"
    Read [Tutorial 2](rigid-mlip.md) first — this tutorial shows only the settings that differ from the baseline configuration.

---

## Quick Start

The ready-to-run example is in `examples/04_flexible/UJIRIO/`:

1. **Copy the example** (includes a pre-computed `conformers.json`):

    ```bash
    cp -r examples/04_flexible/UJIRIO my_flexible_run
    cd my_flexible_run
    ```

2. **Edit `ui.conf`** — update paths as needed

3. **Edit `submit.sh`** — set your allocation and activate the environment

4. **Submit**: `sbatch submit.sh`

---

## What Makes Flexible Different

Rigid CSP searches over crystal packing only. Flexible CSP simultaneously explores **molecular conformation** and packing:

| Setting | Rigid | Flexible |
|---|---|---|
| `flexible` | `FALSE` | `TRUE` |
| `[conformer]` section | Not needed | Required |
| `crossover_probability` | 0.25 | **0.75** |
| Mutation types | Translation, rotation, strain | + Conformer swap, torsion angle |

`crossover_probability = 0.75` is recommended because crossover blends torsion angles — the primary mechanism for discovering new conformations in a crystal packing context.

---

## Step 1: Prepare the Conformer Pool

The conformer pool is a set of molecular geometries representing distinct low-energy conformations. **20–50 conformers** covering distinct torsion angle combinations is usually sufficient.

=== "Using RDKit"

    ```python
    from rdkit import Chem
    from rdkit.Chem import AllChem
    from ase.io.jsonio import encode
    from ase import Atoms
    import json

    mol = Chem.MolFromMolFile("molecule.sdf")
    mol = Chem.AddHs(mol)
    AllChem.EmbedMultipleConfs(mol, numConfs=200, pruneRmsThresh=0.5)
    AllChem.MMFFOptimizeMoleculeConfs(mol)

    conformers = {}
    for i, conf in enumerate(mol.GetConformers()):
        pos = conf.GetPositions()
        symbols = [atom.GetSymbol() for atom in mol.GetAtoms()]
        atoms = Atoms(symbols=symbols, positions=pos)
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformers.json", "w") as f:
        json.dump(conformers, f)
    ```

=== "Using CREST"

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json

    conformers = {}
    atoms_list = read("crest_conformers.xyz", index=":")
    for i, atoms in enumerate(atoms_list):
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformers.json", "w") as f:
        json.dump(conformers, f)
    ```

=== "Pre-computed (UJIRIO)"

    A pre-computed `conformers.json` is included in the example — no preparation needed.

---

## Step 2: Configuration Changes

Starting from the [baseline config](rigid-mlip.md#configuration-reference), add and change these sections:

```ini
[run_settings]
num_molecules = 4
flexible = TRUE                              # Enable conformer-aware operators
end_ga_structures_added = 500

[conformer]
pre_calc_conformer_json_name = conformers.json
boltzmann_scale_factor = 1                   # Higher = flatter conformer distribution
conformer_selection_type = mean_fitness      # Weights conformer choice by pool energy
volume_mult = 1.2                            # Volume multiplier for cell estimation

[crossover]
crossover_probability = 0.75                 # Higher for flexible (torsion blending)

[cell_check_settings]
specific_radius_proportion = 0.79            # From pool analysis for UJIRIO
```

All other sections remain the same as the baseline.

### What Happens During the Run

During initial pool preparation, GAtor will:

1. Optimize each crystal structure
2. Deduplicate conformers using RMSD
3. Assign conformer IDs to each molecule in each crystal
4. Save the deduplicated conformer pool to `unique_mol_dict.json`

---

## Flexible Cocrystal

For flexible cocrystals, provide **separate conformer files** for each molecular type:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 4
stoic = 1 1
num_atoms = 20 15
flexible = TRUE

[conformer]
pre_calc_conformer_json_name = conformer_0.json conformer_1.json
```

---

## Tips

!!! tip "Conformer Diversity"
    GAtor uses Boltzmann-weighted selection for conformer mutations. Increase `boltzmann_scale_factor` for a flatter distribution that samples more diverse conformers.

!!! tip "Very Flexible Molecules"
    For molecules with many rotatable bonds, use CREST for thorough conformer sampling (100–200 conformers).

!!! tip "Restart"
    If the job times out, resubmit — GAtor continues from the existing pool. The conformer pool (`unique_mol_dict.json`) is preserved.

---

## Next Steps

- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
