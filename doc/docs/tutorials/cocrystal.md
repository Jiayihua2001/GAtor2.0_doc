# Tutorial 2: Cocrystal Prediction

This tutorial demonstrates crystal structure prediction for a binary cocrystal — two distinct molecular components packed in the same crystal lattice (e.g., an API + coformer).

---

## Scenario

- **System**: A 1:1 cocrystal (e.g., pharmaceutical API + coformer)
- **Z = 4**: 2 molecules of type A + 2 molecules of type B per unit cell
- **Z' = 1**: One formula unit per asymmetric unit
- **Backend**: UMA on GPU
- **Cluster**: NERSC Perlmutter (adapt for your system)

## What You Need

| File | Description | Required? |
|------|-------------|-----------|
| `structures.json` | Initial cocrystal structures (ASE JSON) | Yes |
| `ui.conf` | Configuration file | Yes |
| `submit_gpu.sh` | SLURM submission script | Yes |

## Step 1: Prepare Cocrystal Structures

Your `structures.json` must contain structures with **both molecular types** and consistent atom ordering.

### Atom Ordering Convention

For a 1:1 cocrystal with Z=4 (2 molecules of A + 2 of B), atoms must be ordered as:

```
[A₁ atoms (20)][B₁ atoms (15)][A₂ atoms (20)][B₂ atoms (15)]
```

Where the numbers in parentheses are the atom count per molecule.

### Converting CIF Files

Use the helper script to convert CIF files to `structures.json`:

```bash
python /path/to/GAtor/example/prepare_structures.py /path/to/cocrystal_cifs \
    --output structures.json
```

Or convert manually:

```python
from ase.io import read
from ase.io.jsonio import encode
import json, glob, os

structures = {}
for cif_file in sorted(glob.glob("cocrystal_cifs/*.cif")):
    atoms = read(cif_file)
    name = os.path.splitext(os.path.basename(cif_file))[0]
    structures[name] = encode(atoms)

with open("structures.json", "w") as f:
    json.dump(structures, f)
```

!!! warning "Atom Ordering"
    If your CIF files have inconsistent atom ordering (e.g., from different structure generators), set `reorder_initial_pool = TRUE` in `[initial_pool]`. GAtor will automatically reorder atoms to match a reference molecule.

## Step 2: Write Configuration

Create `ui.conf`. A complete example is available at `example/cocrystal/MLIP/ui.conf`:

```ini
# ==============================================================================
# GAtor 2.0: Cocrystal CSP with UMA
# ==============================================================================
# System: 1:1 binary cocrystal (e.g., API + coformer)
# Backend: UMA MLIP on GPU
# ==============================================================================

[GAtor_master]
fill_initial_pool = TRUE
run_ga = TRUE

[modules]
initial_pool_module = IP_filling
optimization_module = UMA
spe_module = UMA
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
# --- Cocrystal-specific settings ---
crystal = co-crystal
num_molecules = 4
Z_prime = 1
stoic = 1 1            # 1:1 stoichiometry (1 mol A : 1 mol B per formula unit)
num_atoms = 20 15      # 20 atoms in molecule A, 15 in molecule B
end_ga_structures_added = 1000
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
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.75

[mutation]
stand_dev_trans = 3.0
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
# Set target_volume based on combined molecular volumes * Z
# target_volume = 2500
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

### Key Cocrystal-Specific Settings

| Setting | Section | Description |
|---------|---------|-------------|
| `crystal = co-crystal` | `[run_settings]` | Enables cocrystal mode |
| `stoic = 1 1` | `[run_settings]` | Stoichiometry ratio (A:B) |
| `num_atoms = 20 15` | `[run_settings]` | Atoms per molecule type |
| `Z_prime = 1` | `[run_settings]` | Formula units per asymmetric unit |
| `num_molecules = 4` | `[run_settings]` | Total molecules in unit cell (= sum(stoic) * Z) |

### Crossover Options

GAtor provides two crossover modes for cocrystals:

=== "Symmetric Crossover (Default)"

    Handles each molecular type independently — lattice, COM positions, and orientations are crossed over per component type.

    ```ini
    [modules]
    crossover_module = symmetric_crossover
    ```

=== "Dimer Crossover"

    Treats molecular dimers (A-B pairs) as single units during crossover. Can be useful when the intermolecular interaction between A and B is the dominant packing force.

    ```ini
    [modules]
    crossover_module = crossover_dimers
    ```

## Step 3: Write SLURM Script

Create `submit_gpu.sh` (see `example/cocrystal/MLIP/submit_gpu.sh`):

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH --time=04:00:00
#SBATCH -C gpu
#SBATCH --gpus-per-node=4
#SBATCH -q regular
#SBATCH -A your_account
#SBATCH -J gator_cocrystal
#SBATCH -o gator_%j.out
#SBATCH -e gator_%j.err

ulimit -s unlimited
ulimit -v unlimited

conda activate gator
python /path/to/GAtor/gator/GAtor_master.py ui.conf
```

## Step 4: Submit and Monitor

```bash
sbatch submit_gpu.sh

# Monitor progress
tail -f GAtor.log

# Check energy hierarchy
cat tmp/energy_hierarchy_*.dat
```

## Step 5: Analyze Results

GAtor automatically handles cocrystal-specific processing during the run:

1. **Molecule extraction** — Extracts reference geometries `mol_1.in` and `mol_2.in` from the initial pool
2. **ASU partner swap** — Ensures consistent molecule labeling across all structures
3. **Stoichiometry preservation** — Maintains the correct A:B ratio during crossover and mutation

Results are in the same format as single-component runs:

```bash
# Ranked structures by energy
head -20 tmp/energy_hierarchy_*.dat

# Structure pool
ls tmp/pool_0/
```

---

## Z' > 1 (Single Component)

For single-component systems with Z' > 1 (multiple symmetry-independent molecules), use cocrystal mode with a single-type stoichiometry:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 8       # e.g., Z'=2 with Z=4 → 8 molecules
Z_prime = 2
stoic = 1               # Single component
num_atoms = 30           # Atoms per molecule
```

This allows GAtor to treat each symmetry-independent molecule separately during crossover and mutation.

---

## Tips

!!! tip "Volume Estimation"
    For cocrystals, the target volume should account for both molecular types. If you omit `target_volume`, GAtor estimates it from the initial structures. For manual estimation: `target_volume ≈ (V_mol_A + V_mol_B) * Z / stoic_sum`.

!!! tip "Stoichiometry"
    The `stoic` setting defines the ratio, not absolute counts. For a 2:1 cocrystal (e.g., 2 API + 1 coformer per formula unit), use `stoic = 2 1` with `num_molecules = 6` (for Z'=1, Z=2).

!!! tip "Restart"
    If the job times out, resubmit with `restart_replicas = TRUE`. GAtor resumes from the existing pool and maintains all cocrystal metadata.
