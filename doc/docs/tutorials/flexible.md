# Tutorial 3: Flexible Molecule

This tutorial covers crystal structure prediction for molecules with conformational flexibility, such as pharmaceuticals with rotatable bonds.

---

## Scenario

- **System**: A flexible organic molecule with rotatable bonds
- **Z = 4**: 4 molecules per unit cell
- **Backend**: MACE on GPU
- **Conformers**: Pre-computed set of molecular conformers
- **Cluster**: NERSC Perlmutter (adapt for your system)

## What You Need

| File | Description | Required? |
|------|-------------|-----------|
| `structures.json` | Initial crystal structures (ASE JSON) | Yes |
| `conformer.json` | Pre-computed molecular conformers | Yes |
| `ui.conf` | Configuration file | Yes |
| `submit_gpu.sh` | SLURM submission script | Yes |

## Step 1: Prepare the Conformer Pool

The conformer pool is a set of molecular geometries representing distinct low-energy conformations. GAtor uses this pool for **conformer mutation** (swapping the molecular shape in a crystal) and **torsion angle crossover** (blending dihedral angles between parents).

=== "Using RDKit (Recommended)"

    GAtor provides a helper script at `example/flexible/prepare_conformers.py`:

    ```bash
    # From an SDF file containing your molecule
    python /path/to/GAtor/example/flexible/prepare_conformers.py molecule.sdf \
        --num_conformers 200 \
        --output conformer.json
    ```

    This uses RDKit's ETKDG method to generate diverse 3D conformers.

=== "Using CREST"

    Run CREST for conformer sampling, then convert to GAtor format:

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json

    conformers = {}
    atoms_list = read("crest_conformers.xyz", index=":")
    for i, atoms in enumerate(atoms_list):
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformer.json", "w") as f:
        json.dump(conformers, f)
    ```

=== "Manual Conversion"

    Convert any set of molecular geometries (XYZ, SDF, etc.):

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json, glob

    conformers = {}
    for i, xyz_file in enumerate(sorted(glob.glob("conformers/*.xyz"))):
        atoms = read(xyz_file)
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformer.json", "w") as f:
        json.dump(conformers, f)
    ```

!!! important "Conformer Energies"
    GAtor will **re-compute** conformer energies using the configured optimization module (MACE in this case) during pool preparation. This ensures consistency between crystal and molecular energies. You do not need to pre-compute energies.

!!! tip "How Many Conformers?"
    Aim for **50–200 conformers** covering distinct torsion angle combinations. GAtor de-duplicates conformers using the `rmsd_threshold` parameter, so it's better to over-generate.

## Step 2: Prepare Initial Structures

Prepare `structures.json` as described in the [Quick Start](../getting-started/quickstart.md#generating-structuresjson). The same methods work for flexible molecules.

```bash
python /path/to/GAtor/example/prepare_structures.py /path/to/cif_directory \
    --output structures.json
```

## Step 3: Write Configuration

Create `ui.conf`. A complete example is available at `example/flexible/ui.conf`:

```ini
# ==============================================================================
# GAtor 2.0: Flexible Molecule CSP with MACE
# ==============================================================================

[GAtor_master]
fill_initial_pool = TRUE
run_ga = TRUE

[modules]
initial_pool_module = IP_filling
optimization_module = MACE
spe_module = MACE
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
stored_energy_name = mace_lbfgs
prepare_initial_pool = TRUE

[run_settings]
num_molecules = 4
flexible = TRUE
end_ga_structures_added = 1000
output_all_geometries = TRUE
restart_replicas = TRUE

# --- Conformer Settings ---
[conformer]
pre_calc_conformer_json_name = conformer.json
rmsd_threshold = 1.5

[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4
processes_per_replica = 1
python_command = python

[MACE]
store_energy_names = mace mace_lbfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.75
# Torsion angle blending during crossover
blend_torsion_prob = 0.5

[mutation]
stand_dev_trans = 3.0
stand_dev_rot = 30
stand_dev_strain = 0.3
# Conformer and torsion mutations are automatically included
# when flexible = TRUE. Categories:
#   Translation/Rotation, Permutation/Block, Strain, Conformer
# Conformer category weight is 0.1 by default

[cell_check_settings]
# Set target_volume based on your molecule's vdW volume * Z
# target_volume = 1800
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.65
full_atomic_distance_check = 0.1

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
|---------|---------|-------------|
| `flexible = TRUE` | `[run_settings]` | Enables conformer-aware operators |
| `pre_calc_conformer_json_name` | `[conformer]` | Path to conformer pool file |
| `rmsd_threshold` | `[conformer]` | RMSD threshold (A) for conformer deduplication |
| `blend_torsion_prob` | `[crossover]` | Probability of torsion angle blending in crossover |

## Step 4: Write SLURM Script

Create `submit_gpu.sh` (see `example/flexible/submit_gpu.sh`):

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH --time=04:00:00
#SBATCH -C gpu
#SBATCH --gpus-per-node=4
#SBATCH -q regular
#SBATCH -A your_account
#SBATCH -J gator_flexible
#SBATCH -o gator_%j.out
#SBATCH -e gator_%j.err

ulimit -s unlimited
ulimit -v unlimited

conda activate gator
python /path/to/GAtor/gator/GAtor_master.py ui.conf
```

## Step 5: Submit and Monitor

```bash
sbatch submit_gpu.sh

# Monitor progress
tail -f GAtor.log

# Check energy hierarchy
cat tmp/energy_hierarchy_*.dat
```

## Step 6: Analyze Results

During initial pool preparation, GAtor will:

1. Optimize each crystal structure with MACE
2. Re-compute conformer energies with MACE for consistency
3. Deduplicate conformers using RMSD
4. Assign conformer IDs to each molecule in each crystal
5. Save the deduplicated conformer pool to `unique_mol_dict.json`

Each structure in the pool tracks its conformer assignments:

- `conformer_0_ID` — ID of the assigned conformer for molecule 0
- `conformer_0_energy` — Intramolecular energy of that conformer

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

Where `conformer_0.json` contains conformers for molecule type A and `conformer_1.json` for type B.

---

## Tips

!!! tip "Conformer Selection"
    GAtor uses **Boltzmann-weighted selection** by default when choosing conformers for mutation. Lower-energy conformers are preferred, but higher-energy ones are still sampled to maintain diversity.

!!! tip "Torsion Angle Crossover"
    The `blend_torsion_prob` parameter controls how often torsion angles are blended (vs. swapped) during crossover. Higher values (0.5–0.8) work well for molecules with many rotatable bonds.

!!! tip "Restart"
    If the job times out, resubmit with `restart_replicas = TRUE`. GAtor continues from the existing pool. The conformer pool (`unique_mol_dict.json`) is also preserved.
