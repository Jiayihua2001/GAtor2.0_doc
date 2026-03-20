# Tutorial 2: Rigid Molecule with MLIP

This tutorial walks through a complete crystal structure prediction for **Uracil** — a rigid organic molecule (Z=4) — with two calculator options: UMA (MLIP on GPU) and FHI-aims (DFT on CPU).

---

## Scenario

- **Target**: Uracil (C₄H₄N₂O₂), 12 atoms per molecule
- **Z = 4**: 4 molecules per unit cell
- **Experimental structure**: CSD entry 1278441
- **Backends**: UMA (GPU, fast) or FHI-aims (CPU, accurate)

## Files

The ready-to-run example is in `examples/02_quick_start/Uracil/`:

```
02_quick_start/
└── Uracil/
    ├── MLIP/                         # UMA on GPU (recommended for first runs)
    │   ├── ui.conf                   # GAtor configuration
    │   ├── submit.sh                 # SLURM submit script (GPU, 30 min)
    │   ├── structures.json           # Pre-generated initial pool (Genarris)
    │   └── 1278441.cif              # Experimental structure (validation)
    └── DFT/                          # FHI-aims on CPU
        ├── ui.conf                   # GAtor configuration
        ├── submit.sh                 # SLURM submit script (CPU)
        ├── mol.in                    # Uracil molecule definition (12 atoms)
        ├── aims.json                 # FHI-aims calculator settings
        ├── structures.json           # Pre-generated initial pool (Genarris)
        └── 1278441.cif              # Experimental structure (validation)
```

## Quick Start

1. **Choose a calculator** and copy the example:

    ```bash
    # For MLIP (fast, GPU) — recommended for first runs
    cp -r examples/02_quick_start/Uracil/MLIP my_uracil_run

    # For DFT (accurate, CPU)
    cp -r examples/02_quick_start/Uracil/DFT my_uracil_run
    ```

2. **Edit `ui.conf`** — update `exp_path` to the absolute path of `1278441.cif`:

    ```ini
    [initial_pool]
    exp_path = /absolute/path/to/1278441.cif
    ```

3. **Edit `submit.sh`** — set your allocation and activate your environment:

    ```bash
    #SBATCH -A your_allocation            # <- UPDATE
    # conda activate gator                # <- Uncomment
    ```

    !!! note "DFT Only"
        Update `path_to_aims_executable` in `ui.conf` and `species_dir` in `aims.json` for your system.

4. **Submit and monitor**:

    ```bash
    cd my_uracil_run
    sbatch submit.sh
    tail -f GAtor.log
    ```

## MLIP vs DFT Comparison

| | MLIP (UMA) | DFT (FHI-aims) |
|---|---|---|
| **Speed** | ~30 min | ~48 hours |
| **Hardware** | GPU | CPU |
| **Nodes** | 1 (4 GPUs = 4 replicas) | 2+ (1 replica/node) |
| **Accuracy** | Good for screening | Production quality |
| **Queue** | `debug` (30 min) | `regular` |
| **Extra files** | — | `mol.in`, `aims.json` |

## Configuration Walkthrough

### MLIP Configuration (`Uracil/MLIP/ui.conf`)

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
remove_match = TRUE
exp_path = ./1278441.cif                     # <- UPDATE to absolute path

[run_settings]
num_molecules = 4                            # Z=4 for Uracil
end_ga_structures_added = 200                # GA budget

[parallel_settings]
parallelization_method = srun
replicas_per_node = 4
run_on_gpu = TRUE

[UMA]
store_energy_names = uma uma_lbfgs           # Two-stage: SPE → full optimization
relative_energy_thresholds = 1000 3          # Generous first, tight second (eV)
reject_if_worst_energy = FALSE FALSE
fmax = 0.01                                  # Force convergence threshold
steps = 1500                                 # Max optimization steps

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.25                 # Standard for rigid molecules

[mutation]
stand_dev_trans = 0.3
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.75            # From pool analysis

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

### Key Parameters

| Parameter | Value | Notes |
|---|---|---|
| `num_molecules` | `4` | Z=4 for Uracil |
| `end_ga_structures_added` | `200` | GA budget (increase for production) |
| `crossover_probability` | `0.25` | Standard for rigid molecules |
| `specific_radius_proportion` | `0.75` | From pool analysis |
| `relative_energy_thresholds` | `1000 3` | Two-stage: generous → tight |

### DFT Configuration (`Uracil/DFT/ui.conf`)

The DFT configuration differs primarily in the calculator and parallelization:

```ini
[modules]
optimization_module = FHI-aims

[parallel_settings]
parallelization_method = mpirun
replicas_per_node = 1
run_on_gpu = FALSE

[FHI-aims]
execute_command = mpirun
path_to_aims_executable = path/to/aims.x     # <- UPDATE
aims_settings_path = ./aims.json
k_density = 25
fmax = 0.01
steps = 100
store_energy_names = aims aims_bfgs
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
save_failed_calc = TRUE
```

## What This Demonstrates

- **Basic GA workflow**: initial pool → evolution → convergence
- **Two-stage energy filtering**: generous threshold (1000 eV) after single-point evaluation, tight threshold (3 eV) after full optimization
- **Experimental match tracking**: GAtor logs when a GA-generated structure matches the experimental crystal in `GAtor.log`
- **Test mode**: with `remove_match = TRUE`, the experimental structure is removed from the initial pool but tracked during the GA for validation

## Analyzing Results

```bash
# View ranked structures (lowest energy first)
head -20 tmp/energy_hierarchy_*.dat

# Check for experimental matches
grep "match" GAtor.log

# Check experimental match record
cat tmp/exp_match_record.dat
```

## Adapting for Your Molecule

| Parameter | How to Determine |
|---|---|
| `num_molecules` | Z value from experiment or Genarris |
| `specific_radius_proportion` | Pool analysis ([Tutorial 1](setup.md)) |
| `end_ga_structures_added` | 200–500 for MLIP, 100–200 for DFT |
| `crossover_probability` | 0.25 for rigid, 0.75 for flexible |

---

## Tips

!!! tip "Start with MLIP"
    Start with MLIP (UMA or MACE) to quickly validate your setup and GA parameters. A 30-minute debug run on 1 GPU node can confirm convergence behavior before committing to multi-hour DFT runs.

!!! tip "Convergence"
    Monitor `energy_hierarchy_*.dat` — when the lowest energies stabilize and the top-10 mean plateaus, the GA has likely found the global minimum region.

!!! tip "Restart"
    If the job times out, simply resubmit. GAtor detects existing pool structures and continues from where it left off.

!!! tip "Scaling Up"
    For production runs, use 2–4 GPU nodes (8–16 replicas) and increase `end_ga_structures_added` to 500–2000.

---

## Next Steps

- [Tutorial 3: Cocrystal Prediction](cocrystal.md) — Multi-component crystals
- [Tutorial 4: Flexible Molecule](flexible.md) — Conformational flexibility
- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
