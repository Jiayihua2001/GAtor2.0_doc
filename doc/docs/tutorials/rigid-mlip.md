# Tutorial 2: Rigid Molecule with MLIP

Predict the crystal structure of **Uracil** (C₄H₄N₂O₂, Z=4) — a rigid organic molecule — using UMA on GPU or FHI-aims on CPU.

---

## Quick Start

The ready-to-run example is in `examples/02_quick_start/Uracil/`:

1. **Copy the example**:

    ```bash
    cp -r examples/02_quick_start/Uracil/MLIP my_uracil_run   # GPU (fast)
    # OR
    cp -r examples/02_quick_start/Uracil/DFT my_uracil_run    # CPU (accurate)
    ```

2. **Edit `ui.conf`** — set the experimental structure path:

    ```ini
    [initial_pool]
    exp_path = /absolute/path/to/1278441.cif
    ```

3. **Edit `submit.sh`** — set your allocation and activate the environment:

    ```bash
    #SBATCH -A your_allocation            # <- your SLURM account
    # conda activate gator                # <- uncomment
    ```

4. **Submit**:

    ```bash
    cd my_uracil_run
    sbatch submit.sh
    tail -f GAtor.log
    ```

---

## MLIP vs DFT

| | MLIP (UMA) | DFT (FHI-aims) |
|---|---|---|
| **Speed** | ~30 min | ~48 hours |
| **Hardware** | GPU | CPU |
| **Nodes** | 1 (4 GPUs = 4 replicas) | 2+ (1 replica/node) |
| **Accuracy** | Good for screening | Production quality |
| **Extra files** | — | `mol.in`, `aims.json` |

!!! note "DFT Setup"
    For FHI-aims, also update `path_to_aims_executable` in `ui.conf` and `species_dir` in `aims.json`.

---

## Configuration Reference

This is the **baseline configuration** used by all subsequent tutorials. Tutorials 3–5 show only the sections that differ.

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
num_molecules = 4                            # Z value
end_ga_structures_added = 200                # GA budget

[parallel_settings]
parallelization_method = srun
replicas_per_node = 4
run_on_gpu = TRUE

[UMA]
store_energy_names = uma uma_lbfgs           # Two-stage: SPE → full optimization
relative_energy_thresholds = 1000 3          # Generous first, tight second (eV)
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 1500

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

| Parameter | Value | How to Determine |
|---|---|---|
| `num_molecules` | `4` | Z value from experiment or Genarris |
| `specific_radius_proportion` | `0.75` | Pool analysis ([Tutorial 1](setup.md)) |
| `end_ga_structures_added` | `200` | 200–500 for MLIP, 100–200 for DFT |
| `crossover_probability` | `0.25` | Standard for rigid molecules |
| `relative_energy_thresholds` | `1000 3` | Two-stage: generous SPE → tight after optimization |

### DFT Differences

Only the calculator and parallelization sections change for DFT:

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

---

## Checking Results

```bash
# View ranked structures (lowest energy first)
head -20 tmp/energy_hierarchy_*.dat

# Check for experimental matches
cat tmp/exp_match_record.dat
grep "match" GAtor.log
```

- **`Added = 0`** → initial pool structure; **`Added > 0`** → GA-generated
- With `remove_match = TRUE`, the experimental structure is removed from the initial pool but tracked during the GA for validation

---

## Tips

!!! tip "Start with MLIP"
    A 30-minute debug run on 1 GPU node confirms convergence behavior before committing to multi-hour DFT runs.

!!! tip "Restart"
    If the job times out, simply resubmit — GAtor detects existing pool structures and continues from where it left off.

!!! tip "Scaling Up"
    For production runs, use 2–4 GPU nodes (8–16 replicas) and increase `end_ga_structures_added` to 500–2000.

---

## Next Steps

- [Tutorial 3: Cocrystal Prediction](cocrystal.md) — Multi-component crystals
- [Tutorial 4: Flexible Molecule](flexible.md) — Conformational flexibility
- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
