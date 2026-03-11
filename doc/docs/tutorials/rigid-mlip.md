# Tutorial 1: Rigid Molecule with MLIP

This tutorial walks through a complete crystal structure prediction for a rigid organic molecule using the UMA machine-learning potential on a GPU cluster.

---

## Scenario

- **Target**: A rigid organic molecule (e.g., naphthalene, aspirin)
- **Z = 4**: 4 molecules per unit cell
- **Backend**: UMA (FAIRChem) on GPU
- **Cluster**: NERSC Perlmutter (adapt for your system)

## Step 1: Prepare Input Files

### 1.1 Generate Initial Structures

Use PyGenarris or other tools to generate random crystal structures and save them as `structures.json`:

```python
from ase.io import read
from ase.io.jsonio import encode
import json, glob

structures = {}
for i, cif in enumerate(sorted(glob.glob("generated_structures/*.cif"))):
    atoms = read(cif)
    structures[f"struct_{i:04d}"] = encode(atoms)

with open("structures.json", "w") as f:
    json.dump(structures, f)
```

### 1.2 Create Working Directory

```bash
mkdir naphthalene_csp && cd naphthalene_csp
cp /path/to/structures.json .
```

## Step 2: Write Configuration

Create `ui.conf`:

```ini
#=============================================
# GAtor Configuration - Rigid Molecule + UMA
#=============================================

[GAtor_master]
fill_initial_pool = TRUE
run_ga = TRUE

#--- Module Selection ---
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

#--- Fitness ---
[fitness]
energy_name = energy

#--- Initial Pool ---
[initial_pool]
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
prepare_initial_pool = TRUE

#--- Crystal System ---
[run_settings]
num_molecules = 4
end_ga_structures_added = 500
output_all_geometries = TRUE
restart_replicas = TRUE

#--- Parallelization ---
[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4
processes_per_replica = 1
python_command = python

#--- UMA Settings ---
[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500

#--- Selection ---
[selection]
tournament_size = 10

#--- Crossover ---
[crossover]
crossover_probability = 0.75

#--- Mutation ---
[mutation]
stand_dev_trans = 3.0
stand_dev_rot = 30
stand_dev_strain = 0.3

#--- Structure Validation ---
[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.65
full_atomic_distance_check = 0.1

#--- Duplicate Detection ---
[pre_relaxation_comparison]
ltol = .5
stol = .5
angle_tol = 10

[post_relaxation_comparison]
energy_comp_window = 1.5
ltol = .5
stol = .5
angle_tol = 10

#--- Clustering ---
[clustering]
clustering_algorithm = AffinityPropagation
feature_vector = RSF
```

## Step 3: Write SLURM Script

Create `submit.sh`:

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH --time=04:00:00
#SBATCH -C gpu
#SBATCH --gpus-per-node=4
#SBATCH -q regular
#SBATCH -A your_account
#SBATCH -J naphthalene_csp
#SBATCH -o gator_%j.out
#SBATCH -e gator_%j.err

ulimit -s unlimited
ulimit -v unlimited

# Load environment
conda activate gator

# Run GAtor
python /path/to/GAtor/gator/GAtor_master.py ui.conf
```

## Step 4: Submit and Monitor

```bash
sbatch submit.sh

# Monitor the master log
tail -f GAtor.log

# Check how many structures have been added
wc -l tmp/energy_hierarchy_*.dat
```

## Step 5: Analyze Results

After the run completes (or reaches `end_ga_structures_added = 500`):

```bash
# View ranked structures
head -20 tmp/energy_hierarchy_*.dat

# Count unique space groups
python -c "
import json
with open('tmp/initial_pool_space_groups.json') as f:
    data = json.load(f)
for sg in data['space_groups'][:10]:
    print(f\"SG {sg['space_group_number']} ({sg['space_group_symbol']}): {sg['count']}\")
"
```

The lowest-energy structures in `tmp/pool_0/` are your best predictions. Convert them to CIF for visualization:

```python
from gator.utilities.io import read_json_to_struct

struct = read_json_to_struct("tmp/pool_0/best_structure.json")
struct.get_pymatgen_structure().to(filename="best.cif", fmt="cif")
```

---

## Tips

!!! tip "Convergence"
    Monitor the energy hierarchy file — when the lowest energies stabilize, the GA has likely found the global minimum region.

!!! tip "Restart"
    If the job times out, simply resubmit. GAtor will detect existing pool structures and continue from where it left off (with `restart_replicas = TRUE`).

!!! tip "Scaling Up"
    For production runs, use 2-4 GPU nodes (8-16 replicas) and set `end_ga_structures_added = 2000-5000`.
