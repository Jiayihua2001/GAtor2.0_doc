# Quick Start

This guide walks you through running your first GAtor crystal structure prediction.

---

## Prerequisites

- GAtor is [installed](installation.md) and the `gator` Conda environment is activated
- Access to a GPU node (for MLIP-based runs) or CPU nodes (for DFT-based runs)

## Preparation Checklist

Before running GAtor, ensure you have the following inputs ready:

### Required for All Runs

- [x] **`structures.json`** — Initial crystal structures in ASE JSON format
- [x] **`ui.conf`** — Configuration file defining all GA parameters
- [x] **SLURM submission script** — For launching on HPC clusters

### Additional Requirements by System Type

=== "Rigid Molecule"

    No additional files needed beyond the basics above.

=== "Flexible Molecule"

    - [x] **`conformers.json`** — Pre-computed molecular conformer pool (see [Tutorial 4](../tutorials/flexible.md))

=== "Cocrystal"

    - [x] Structures in `structures.json` must contain **both molecular types** with consistent atom ordering
    - [x] `stoic` and `num_atoms` must be set in `[run_settings]`
    - [x] `mol_1.in` and `mol_2.in` — Molecule definition files (auto-extracted if `remove_match = TRUE`)

### Optional

- [ ] **Experimental CIF** — For removing known structures from the pool (`remove_match = TRUE`)
- [ ] **Experimental PXRD pattern** (`.xy` file) — For PXRD-assisted fitness (`fitness_module = vc_energy_fitness`)
- [ ] **critic2** — Required only if using PXRD fitness (see [Installation](installation.md#optional-critic2-for-pxrd-fitness))

---

## Step 1: Prepare the Initial Pool

GAtor needs an initial set of crystal structures. We recommend generating the initial pool from [Genarris 3.0](https://github.com/Yi5817/Genarris). For other sources, use the provided helper script:

```bash
# From CIF files
python examples/01_prepare/prepare_structures.py /path/to/cif_directory --output structures.json

# From POSCAR files
python examples/01_prepare/prepare_structures.py /path/to/poscar_directory --format poscar --output structures.json

# From Genarris output (already in ASE JSON format)
python examples/01_prepare/prepare_structures.py genarris_output.json --format genarris --output structures.json
```

!!! tip "What is `structures.json`?"
    A JSON dictionary mapping structure IDs to ASE Atoms objects serialized via `ase.io.jsonio.encode()`. GAtor reads, optimizes, and filters these structures to form the initial pool.

## Step 2: Pool Analysis — Determine SR

The **specific radius proportion** (SR) controls how close atoms from different molecules are allowed to be. This is the most critical parameter to get right.

!!! important "Getting SR Right"
    Too high → rejects valid structures → slow convergence.
    Too low → accepts unphysical structures → wasted computation.

Run the pool analysis:

```bash
python examples/01_prepare/analyze_pool.py \
    --ui-conf ui.conf \
    --pool-path ./structures.json \
    --pool-format structures_json \
    --energy-key <energy-key> \
    --output-dir pool_analysis_results \
    --exp-file /path/to/experimental.cif   # optional
```

Then auto-recommend SR:

```bash
python examples/01_prepare/recommend_sr.py pool_analysis_results/pool_analysis.csv
```

**Rule of thumb**: Pick the SR where ~80% of the pool passes. Typical values: 0.75–0.82.

See the [Setup & Preparation Tutorial](../tutorials/setup.md) for a detailed walkthrough.

## Step 3: Create a Configuration File

Create a `ui.conf` file. Here is a minimal example for a rigid molecule with UMA:

```ini
[GAtor_master]
analyze_initial_pool = TRUE

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
remove_match = TRUE
exp_path = ./experimental.cif

[run_settings]
num_molecules = 4
end_ga_structures_added = 200
output_all_geometries = TRUE

[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4

[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 1500
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.25

[mutation]
stand_dev_trans = 0.3
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.75

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

!!! tip "Full Template"
    A comprehensive template with all options is available at `examples/00_setup/ui_template.conf`.

## Step 4: Create a SLURM Submission Script

=== "GPU (MLIP: UMA, MACE, AIMNet2)"

    ```bash
    #!/bin/bash
    #SBATCH -N 2                          # GPU nodes (2 nodes = 8 replicas)
    #SBATCH --time=12:00:00
    #SBATCH -C gpu
    #SBATCH --gpus-per-node=4
    #SBATCH -q regular
    #SBATCH -A your_allocation            # <- UPDATE
    #SBATCH -J gator_mlip

    export SLURM_OVERLAP=1
    export OMP_NUM_THREADS=1
    export MKL_NUM_THREADS=1
    export NUMEXPR_NUM_THREADS=1
    export HF_HUB_OFFLINE=1

    ulimit -s unlimited
    ulimit -v unlimited

    # module load ...                     # <- Load your MPI module
    # conda activate gator                # <- Activate environment

    run_gator ui.conf
    ```

=== "CPU (DFT: FHI-aims, VASP)"

    ```bash
    #!/bin/bash
    #SBATCH -N 2                          # CPU nodes
    #SBATCH --time=48:00:00
    #SBATCH -C cpu
    #SBATCH -q regular
    #SBATCH -A your_allocation            # <- UPDATE
    #SBATCH -J gator_dft

    export SLURM_OVERLAP=1
    export OMP_NUM_THREADS=1
    export MKL_NUM_THREADS=1
    export NUMEXPR_NUM_THREADS=1

    ulimit -s unlimited
    ulimit -v unlimited

    # module load ...                     # <- Load your MPI module
    # conda activate gator                # <- Activate environment

    run_gator ui.conf
    ```

## Step 5: Submit and Monitor

```bash
# Submit the job
sbatch submit.sh

# Monitor progress
tail -f GAtor.log

# Check how many structures have been added
wc -l tmp/energy_hierarchy_*.dat
```

## Step 6: Analyze Results

After the GA completes, your results are in the working directory:

```
my_run/
├── ui.conf                          # Your configuration
├── structures.json                  # Input structures
├── initial_pool/                    # Optimized initial structures
├── structures/                      # Full structure details + metadata
├── tmp/
│   ├── energy_hierarchy_*.dat       # Ranked structures by energy
│   ├── exp_match_record.dat         # Experimental match tracking
│   ├── xtal_fitness.json            # Fitness values per structure
│   └── ...
├── GAtor.log                        # Main log file
└── output/                          # Replica outputs
```

The lowest-energy structures in `energy_hierarchy_*.dat` are your best predictions.

See the [Post-Analysis Tutorial](../tutorials/post-analysis.md) for convergence plots, structure extraction, and visualization.

---

## What's Next?

| | |
|---|---|
| [**Setup & Preparation**](../tutorials/setup.md) | Detailed pool preparation and SR analysis |
| [**Tutorials**](../tutorials/index.md) | Step-by-step examples for every crystal type |
| [**Configuration Guide**](../user-guide/configuration.md) | Understand every section of `ui.conf` |
| [**Parallelization**](../user-guide/parallelization.md) | Scale to multi-node runs |
| [**Post-Analysis**](../tutorials/post-analysis.md) | Analyze and visualize your results |
