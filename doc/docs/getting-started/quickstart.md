# Quick Start

This guide walks you through running your first GAtor crystal structure prediction using the UMA machine-learning potential on a GPU cluster.

---

## Prerequisites

- GAtor is [installed](installation.md) and the `gator` Conda environment is activated
- Access to a GPU node (for MLIP-based runs) or CPU nodes (for DFT-based runs)

## Step 1: Prepare Your Working Directory

Create a fresh directory for your prediction run:

```bash
mkdir my_first_run && cd my_first_run
```

## Step 2: Prepare the Initial Pool

GAtor needs an initial set of crystal structures. The easiest approach is to provide a `structures.json` file containing ASE Atoms objects in JSON format.

You can generate initial structures using tools like **PyGenarris** (included with GAtor), the **CSD** (Cambridge Structural Database), or other structure generators.

```bash
# Example: copy structures from your data source
cp /path/to/your/structures.json .
```

!!! tip "What is `structures.json`?"
    This file is a JSON dictionary mapping structure IDs to ASE Atoms objects serialized via `ase.io.jsonio.encode()`. GAtor will read, optimize, and filter these structures to form the initial pool.

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
