# Tutorial 4: Flexible Molecule

This tutorial covers crystal structure prediction for **UJIRIO** — a molecule with rotatable bonds, using conformer-aware mutations and torsion angle blending.

---

## Scenario

- **System**: UJIRIO — semi-flexible organic with rotatable bonds
- **Z = 4**: 4 molecules per unit cell
- **Backend**: UMA on GPU
- **Conformers**: Pre-computed conformer pool (`conformers.json`)

## Files

The ready-to-run example is in `examples/04_flexible/UJIRIO/`:

```
04_flexible/
└── UJIRIO/
    ├── ui.conf              # Flexible molecule configuration
    ├── submit.sh            # SLURM submit script (GPU)
    ├── structures.json      # Pre-generated initial pool
    └── conformers.json      # Pre-computed conformer pool
```

## What Makes Flexible Different

Rigid CSP searches over crystal packing only. Flexible CSP simultaneously explores **molecular conformation** and packing via:

1. **Conformer pool** (`conformers.json`) — pre-computed low-energy geometries
2. **Conformer swap mutations** — replace molecular geometry with a different conformer
3. **Torsion angle mutations** — perturb individual rotatable bonds
4. **Torsion blending in crossover** — interpolate parent torsion angles

| Setting | Rigid | Flexible |
|---|---|---|
| `flexible` | `FALSE` | `TRUE` |
| `[conformer]` section | Not needed | Required |
| `crossover_probability` | 0.25 | **0.75** |
| `stand_dev_trans` | 0.3 | 0.3 |
| Mutation types | Translation, rotation, strain | + Conformer swap, torsion angle |

!!! note "Higher Crossover Probability"
    `crossover_probability = 0.75` is recommended for flexible molecules because crossover blends torsion angles — the primary mechanism for discovering new conformations in a crystal packing context.

## Step 1: Prepare the Conformer Pool

The conformer pool is a set of molecular geometries representing distinct low-energy conformations. GAtor uses this pool for **conformer mutation** and **torsion angle crossover**.

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

    A pre-computed `conformers.json` is included in the example for UJIRIO — no preparation needed.

!!! tip "How Many Conformers?"
    **20–50 conformers** covering distinct torsion angle combinations is usually sufficient. GAtor de-duplicates using RMSD, so it's better to over-generate.

## Step 2: Prepare Initial Structures

Prepare `structures.json` as described in [Tutorial 1](setup.md):

```bash
python examples/01_prepare/prepare_structures.py /path/to/cif_directory \
    --output structures.json
```

## Step 3: Configuration Walkthrough

```ini
[GAtor_master]
fill_initial_pool = TRUE
run_ga = TRUE

[modules]
initial_pool_module = IP_filling
optimization_module = UMA
comparison_module = structure_comparison
selection_module = Adaptive_tournament_selection
mutation_module = standard_mutation
crossover_module = symmetric_crossover
clustering_module = cluster
fitness_module = standard_energy

[fitness]
energy_name = energy

[initial_pool]
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
prepare_initial_pool = TRUE

[conformer]
# Pre-computed conformer pool
pre_calc_conformer_json_name = conformers.json
# Boltzmann factor for conformer selection (higher = flatter distribution)
boltzmann_scale_factor = 1
# Selection type: 'mean_fitness' weights conformer choice by pool energy
conformer_selection_type = mean_fitness
# Volume multiplier for conformer-based unit cell estimation
volume_mult = 1.2

[run_settings]
num_molecules = 4
flexible = TRUE                      # Enable conformer-aware operators
end_ga_structures_added = 500
output_all_geometries = TRUE

[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4

[UMA]
store_energy_names = uma uma_lbfgs
# Generous first threshold — flexible molecules have large energy spread before relaxation
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 1500
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
# Higher crossover probability for flexible molecules!
# Crossover blends torsion angles — the primary way the GA discovers
# new conformations in a crystal packing context.
crossover_probability = 0.75

[mutation]
# Smaller translation perturbation — conformational degrees of freedom
# already provide ample diversity
stand_dev_trans = 0.3
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.79    # From pool analysis for UJIRIO

[pre_relaxation_comparison]
ltol = .5
stol = .5
angle_tol = 10

[post_relaxation_comparison]
energy_comp_window = 1.5
ltol = .5
stol = .5
angle_tol = 10

[clustering]
clustering_algorithm = AffinityPropagation
feature_vector = RSF
```

### Key Flexible-Specific Settings

| Setting | Section | Description |
|---|---|---|
| `flexible = TRUE` | `[run_settings]` | Enables conformer-aware operators |
| `pre_calc_conformer_json_name` | `[conformer]` | Path to conformer pool file |
| `boltzmann_scale_factor` | `[conformer]` | Controls diversity of conformer selection |
| `conformer_selection_type` | `[conformer]` | How conformers are selected for mutation |
| `volume_mult` | `[conformer]` | Volume multiplier for cell estimation |
| `crossover_probability = 0.75` | `[crossover]` | Higher than rigid (torsion blending) |

## Analyzing Results

During initial pool preparation, GAtor will:

1. Optimize each crystal structure with UMA
2. Re-compute conformer energies for consistency
3. Deduplicate conformers using RMSD
4. Assign conformer IDs to each molecule in each crystal
5. Save the deduplicated conformer pool to `unique_mol_dict.json`

Each structure in the pool tracks its conformer assignments:

```bash
# View the conformer pool
python -c "
import json
with open('unique_mol_dict.json') as f:
    pool = json.load(f)
print(f'Unique conformers: {len(pool)}')
"

# View ranked structures
head -20 tmp/energy_hierarchy_*.dat
```

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

Where `conformer_0.json` contains conformers for molecule type A and `conformer_1.json` for type B.

---

## Tips

!!! tip "Conformer Diversity"
    GAtor uses **Boltzmann-weighted selection** when choosing conformers for mutation. Lower-energy conformers are preferred, but higher-energy ones are still sampled. Increase `boltzmann_scale_factor` for a flatter distribution.

!!! tip "Torsion Angle Crossover"
    During crossover, torsion angles from both parents are blended to create offspring with intermediate conformations. This is controlled by `crossover_probability`.

!!! tip "Very Flexible Molecules"
    For molecules with many rotatable bonds, consider using CREST for thorough conformer sampling (100–200 conformers).

!!! tip "Restart"
    If the job times out, resubmit — GAtor continues from the existing pool. The conformer pool (`unique_mol_dict.json`) is preserved.

---

## Next Steps

- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
