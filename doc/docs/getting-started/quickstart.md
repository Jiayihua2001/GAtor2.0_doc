# Quick Start

This guide walks you through running your first GAtor crystal structure prediction using the UMA machine-learning potential on a GPU cluster.

---

## Prerequisites

- GAtor is [installed](installation.md) and the `gator` Conda environment is activated
- Access to a GPU node (for MLIP-based runs) or CPU nodes (for DFT-based runs)

## Preparation Checklist

Before running GAtor, ensure you have the following inputs ready. The exact requirements depend on your system type:

### Required for All Runs

- [x] **`structures.json`** — Initial crystal structures in ASE JSON format
- [x] **`ui.conf`** — Configuration file defining all GA parameters
- [x] **SLURM submission script** — For launching on HPC clusters

### Additional Requirements by System Type

=== "Rigid Molecule"

    No additional files needed beyond the basics above.

=== "Flexible Molecule"

    - [x] **`conformer.json`** — Pre-computed molecular conformer pool (see [Conformer Preparation](#preparing-a-conformer-pool))

=== "Cocrystal"

    - [x] Structures in `structures.json` must contain **both molecular types** with consistent atom ordering
    - [x] `stoic` and `num_atoms` must be set in `[run_settings]`

### Optional

- [ ] **Experimental CIF** — For removing known structures from the pool (`remove_match = TRUE`)
- [ ] **Experimental PXRD pattern** (`.xy` file) — For PXRD-assisted fitness (`fitness_module = vc_energy_fitness`)
- [ ] **critic2** — Required only if using PXRD fitness (see [Installation](installation.md#optional-critic2-for-pxrd-fitness))
- [ ] **RDKit** — Useful for generating conformer pools (see [Installation](installation.md#optional-rdkit-for-conformer-generation))

---

## Step 1: Prepare Your Working Directory

Create a fresh directory for your prediction run:

```bash
mkdir my_first_run && cd my_first_run
```

## Step 2: Prepare the Initial Pool

GAtor needs an initial set of crystal structures. Provide a `structures.json` file containing ASE Atoms objects in JSON format.

### Generating `structures.json`

You can generate initial structures using **PyGenarris** (included with GAtor), the **CSD**, or other tools, then convert them:

=== "From CIF files"

    Use the helper script provided in `example/prepare_structures.py`:

    ```bash
    python /path/to/GAtor/example/prepare_structures.py /path/to/cif_directory --output structures.json
    ```

    Or manually:

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json, glob, os

    structures = {}
    for cif_file in sorted(glob.glob("my_cifs/*.cif")):
        atoms = read(cif_file)
        name = os.path.splitext(os.path.basename(cif_file))[0]
        structures[name] = encode(atoms)

    with open("structures.json", "w") as f:
        json.dump(structures, f)
    ```

=== "From POSCAR files"

    ```bash
    python /path/to/GAtor/example/prepare_structures.py /path/to/poscar_directory \
        --format poscar --output structures.json
    ```

=== "From PyGenarris"

    If you used Genarris 3.0, the output is already in ASE JSON format:

    ```bash
    python /path/to/GAtor/example/prepare_structures.py genarris_output.json \
        --format genarris --output structures.json
    ```

!!! tip "What is `structures.json`?"
    This file is a JSON dictionary mapping structure IDs to ASE Atoms objects serialized via `ase.io.jsonio.encode()`. GAtor will read, optimize, and filter these structures to form the initial pool.

### Preparing a Conformer Pool

**Required only for flexible molecule runs** (`flexible = TRUE` in `[run_settings]`).

A conformer pool is a set of molecular geometries representing different low-energy conformations. GAtor uses this pool for conformer mutation and torsion angle crossover operators.

=== "Using RDKit (Recommended)"

    Use the helper script provided in `example/flexible/prepare_conformers.py`:

    ```bash
    python /path/to/GAtor/example/flexible/prepare_conformers.py molecule.sdf \
        --num_conformers 200 --output conformer.json
    ```

    This generates conformers using RDKit's ETKDG method and saves them in ASE JSON format.

=== "Using CREST"

    Run CREST to generate conformers, then convert the output:

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json

    # CREST outputs conformers in crest_conformers.xyz
    conformers = {}
    atoms_list = read("crest_conformers.xyz", index=":")
    for i, atoms in enumerate(atoms_list):
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformer.json", "w") as f:
        json.dump(conformers, f)
    ```

=== "Manual"

    Any set of molecular geometries in XYZ/SDF format can be converted:

    ```python
    from ase.io import read
    from ase.io.jsonio import encode
    import json

    conformers = {}
    for i, xyz_file in enumerate(sorted(xyz_files)):
        atoms = read(xyz_file)
        conformers[f"conf_{i:03d}"] = encode(atoms)

    with open("conformer.json", "w") as f:
        json.dump(conformers, f)
    ```

!!! important "Conformer Diversity"
    Aim for 50–200 conformers covering distinct torsion angle combinations. GAtor will de-duplicate conformers using the `rmsd_threshold` parameter.

### Preparing Cocrystal Structures

**Required only for cocrystal runs** (`crystal = co-crystal` in `[run_settings]`).

Structures in `structures.json` must contain **both molecular types** with consistent atom ordering. For a 1:1 cocrystal with Z=4 (2 molecules of type A + 2 of type B):

```
Atoms order: [A₁ atoms][B₁ atoms][A₂ atoms][B₂ atoms]
```

Set the per-molecule atom counts in `ui.conf`:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 4
stoic = 1 1          # 1:1 stoichiometry
num_atoms = 20 15    # 20 atoms in mol A, 15 atoms in mol B
```

!!! tip "Atom Reordering"
    If your input structures have inconsistent atom ordering, set `reorder_initial_pool = TRUE` in `[initial_pool]` to let GAtor fix it automatically.

## Step 3: Create a Configuration File

Create a `ui.conf` file. Here is a minimal example for a rigid molecule with UMA:

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

[run_settings]
num_molecules = 4
end_ga_structures_added = 500
output_all_geometries = TRUE
restart_replicas = TRUE

[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4
processes_per_replica = 1
python_command = python

[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.75

[mutation]
stand_dev_trans = 3.0
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
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

!!! note "Volume Estimation"
    If you omit `target_volume` from `[cell_check_settings]`, GAtor will automatically estimate it from the molecular geometry.

## Step 4: Create a SLURM Submission Script

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH --time=02:00:00
#SBATCH -C gpu
#SBATCH --gpus-per-node=4
#SBATCH -q regular
#SBATCH -A your_account
#SBATCH -J gator_run

ulimit -s unlimited
ulimit -v unlimited

# Activate environment
conda activate gator
# Or: source /path/to/your/env_file.sh

# Launch GAtor
python /path/to/GAtor/gator/GAtor_master.py ui.conf
```

## Step 5: Submit and Monitor

```bash
# Submit the job
sbatch submit.sh

# Monitor progress
tail -f GAtor.log

# Check the energy hierarchy
cat tmp/energy_hierarchy_*.dat
```

## Step 6: Analyze Results

After the GA completes, your results are in the `tmp/` directory:

```
my_first_run/
├── ui.conf                          # Your configuration
├── structures.json                  # Input structures
├── initial_pool/                    # Optimized initial structures
├── tmp/
│   ├── energy_hierarchy_*.dat       # Ranked structures by energy
│   ├── pool_0/                      # Structure pool (JSON files)
│   ├── initial_pool_space_groups.json
│   └── ...
├── GAtor.log                        # Main log file
└── output/                          # Replica outputs
```

The lowest-energy structures in `energy_hierarchy_*.dat` are your best predictions.

---

## What's Next?

- [Configuration Guide](../user-guide/configuration.md) — Understand every section of `ui.conf`
- [Tutorials](../tutorials/index.md) — Step-by-step examples for different scenarios
- [Parallelization](../user-guide/parallelization.md) — Scale to multi-node runs
