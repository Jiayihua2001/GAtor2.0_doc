# Tutorial 1: Setup & Preparation

This tutorial covers HPC configuration, initial pool preparation, and the critical **specific radius proportion** (SR) analysis — everything you need before launching a GA run.

---

## Overview

Before running GAtor, you need to:

1. Choose an energy calculator and configure parallelization
2. Prepare initial crystal structures as `structures.json`
3. Run pool analysis to determine the optimal SR parameter

## Step 1: Choose a Calculator

GAtor supports five energy calculators. Your choice determines hardware requirements and runtime:

| Calculator | Type | Speed | Hardware | Install |
|---|---|---|---|---|
| **UMA** | MLIP | ~30 min | GPU | `pip install -e .[uma] --no-build-isolation` |
| **MACE** | MLIP | ~30 min | GPU | `pip install -e .[mace] --no-build-isolation` |
| **AIMNet2** | MLIP | ~30 min | GPU | `pip install -e .[aimnet2] --no-build-isolation` |
| **FHI-aims** | DFT | 1–5 hours | CPU | External binary |
| **VASP** | DFT | 1–5 hours | CPU | External binary |

!!! tip "Recommendation"
    Start with an **MLIP calculator** (UMA or MACE) for rapid prototyping, then switch to DFT for production-quality results if needed.

### Calculator Configuration

Set the calculator in `ui.conf`:

```ini
[modules]
optimization_module = UMA   # Options: UMA, MACE, AIMNET2, FHI-aims, VASP
```

Each calculator has its own configuration section. See `examples/00_setup/calculator.conf` for complete reference. Here is the UMA example:

```ini
[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 1500
save_trajectory = FALSE
```

**Two-stage energy filtering:** GAtor evaluates structures in two stages. The first threshold (e.g., 1000 eV) is a generous filter after a single-point energy calculation; the second (e.g., 3 eV) is applied after full geometry optimization. This avoids wasting compute on clearly bad structures while keeping valid candidates.

## Step 2: Configure Parallelization

### GPU Clusters (MLIP)

Each GPU provides one GA replica. A typical GPU node has 4 GPUs = 4 replicas.

```ini
[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4
```

| Nodes | Replicas | Use Case |
|---|---|---|
| 1 | 4 | Quick test (`-q debug`, `--time=00:30:00`) |
| 2 | 8 | Standard production run |
| 4 | 16 | Large-scale exploration |

### CPU Clusters (DFT)

DFT is CPU-bound. Each replica gets many CPUs for the ab initio calculation.

```ini
[parallel_settings]
parallelization_method = srun
run_on_gpu = FALSE
replicas_per_node = 1
cpus_per_replica = 128
```

### Submit Scripts

Ready-to-use SLURM scripts are in `examples/00_setup/`:

=== "GPU (MLIP)"

    ```bash
    #!/bin/bash
    #SBATCH -N 2                          # 2 GPU nodes = 8 replicas
    #SBATCH --time=12:00:00
    #SBATCH -C gpu
    #SBATCH --gpus-per-node=4
    #SBATCH -q regular
    #SBATCH -A your_allocation            # <- UPDATE

    export SLURM_OVERLAP=1
    export OMP_NUM_THREADS=1
    export MKL_NUM_THREADS=1
    export NUMEXPR_NUM_THREADS=1
    export HF_HUB_OFFLINE=1

    ulimit -s unlimited
    ulimit -v unlimited

    # module load ...                     # <- Load MPI module
    # conda activate gator                # <- Activate environment

    run_gator ui.conf
    ```

=== "CPU (DFT)"

    ```bash
    #!/bin/bash
    #SBATCH -N 2
    #SBATCH --time=48:00:00
    #SBATCH -C cpu
    #SBATCH -q regular
    #SBATCH -A your_allocation            # <- UPDATE

    export SLURM_OVERLAP=1
    export OMP_NUM_THREADS=1
    export MKL_NUM_THREADS=1
    export NUMEXPR_NUM_THREADS=1

    ulimit -s unlimited
    ulimit -v unlimited

    # module load ...                     # <- Load MPI module
    # conda activate gator                # <- Activate environment

    run_gator ui.conf
    ```

## Step 3: Prepare `structures.json`

### Using Genarris (Recommended)

We recommend generating the initial pool from [Genarris 3.0](https://github.com/Yi5817/Genarris). The output is already in the required format:

```bash
cp /path/to/genarris/output/structures.json ./structures.json
```

### From CIF or POSCAR Files

Use the helper script in `examples/01_prepare/`:

```bash
# From CIF files
python examples/01_prepare/prepare_structures.py /path/to/cif_directory \
    --output structures.json

# From POSCAR files
python examples/01_prepare/prepare_structures.py /path/to/poscar_directory \
    --format poscar --output structures.json
```

### With Atom Reordering (Cocrystals & Z' > 1)

For multi-component systems, consistent atom ordering is critical:

```bash
python examples/01_prepare/prepare_structures.py /path/to/cifs \
    --output structures.json --reorder --ui-conf /path/to/ui.conf
```

!!! note "Cocrystal Atom Ordering"
    For a 1:1 cocrystal with Z=4, atoms must be ordered as:
    `[A₁ atoms][B₁ atoms][A₂ atoms][B₂ atoms]`.
    Use `reorder_initial_pool = TRUE` in `[initial_pool]` to let GAtor fix ordering automatically.

## Step 4: Pool Analysis — Determine SR

The **specific radius proportion** (SR) is the most critical parameter in GAtor. It defines the minimum allowed distance between atoms of different molecules as a fraction of their van der Waals radii.

### Run the Analysis

```bash
python examples/01_prepare/analyze_pool.py \
    --ui-conf ui.conf \
    --pool-path ./structures.json \
    --pool-format structures_json \
    --energy-key <energy-key> \
    --output-dir pool_analysis_results \
    --exp-file /path/to/experimental.cif   # optional
```

**Arguments:**

| Argument | Description |
|---|---|
| `--ui-conf` | Path to your `ui.conf` (needed for molecule settings) |
| `--pool-path` | Path to `structures.json` or structure directory |
| `--pool-format` | `structures_json`, `json_dir`, or `initial_pool` |
| `--energy-key` | Property key for energy (e.g., `uma_lbfgs`, `mace_bfgs`) |
| `--exp-file` | Path to experimental CIF file (optional, for reference markers) |

### Output

The analysis produces:

- **`pool_analysis.csv`** — Per-structure SR, energy, volume, space group, lattice parameters
- **`pool_analysis.png`** — 4-panel figure:
    1. SR histogram — distribution of SR values across the pool
    2. Energy vs Volume — colored by SR
    3. Space group distribution — bar chart
    4. Energy histogram — distribution of energies

### Auto-Recommend SR

```bash
python examples/01_prepare/recommend_sr.py pool_analysis_results/pool_analysis.csv
```

Options:

```bash
# Custom target pass rate (default: 0.80)
python examples/01_prepare/recommend_sr.py pool_analysis.csv --target-pass-rate 0.75

# With known experimental SR
python examples/01_prepare/recommend_sr.py pool_analysis.csv --exp-sr 0.82
```

### Choosing SR Manually

**Rule of thumb**: Pick the SR where ~80% of the pool passes.

| Pool Pass Rate at SR | Action |
|---|---|
| > 95% | SR may be too low — consider increasing |
| 75–90% | Good range for most systems |
| < 60% | SR is too high — lower it or check your pool |

!!! important "Experimental Structure SR"
    If you have an experimental structure, ensure your chosen SR is **at or below** the experimental SR. Otherwise the GA cannot find the experimental structure.

### Minimal `ui.conf` for Pool Analysis

You only need a few sections to run pool analysis:

```ini
[modules]
optimization_module = UMA

[run_settings]
num_molecules = 4

[parallel_settings]
parallelization_method = serial
replicas_per_node = 1

[cell_check_settings]
specific_radius_proportion = 0.75
```

## Step 5: Set Up Comparative Runs (Optional)

Use `setup_runs.py` to create multiple runs with different fitness settings for comparison:

```bash
python examples/00_setup/setup_runs.py setup --base-dir /path/to/base/run \
    --run-names energy_0.25 energy_0.75 niching_0.25 niching_0.75
```

This creates subdirectories with identical inputs but different GA settings, allowing you to compare convergence across strategies.

---

## Next Steps

Choose your crystal type and proceed to the corresponding tutorial:

| Crystal Type | Tutorial |
|---|---|
| Rigid molecule | [Tutorial 2: Rigid Molecule with MLIP](rigid-mlip.md) |
| Multi-component cocrystal | [Tutorial 3: Cocrystal Prediction](cocrystal.md) |
| Flexible molecule | [Tutorial 4: Flexible Molecule](flexible.md) |
| PXRD-guided prediction | [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) |
