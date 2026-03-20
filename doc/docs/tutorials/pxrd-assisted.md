# Tutorial 4: PXRD-Assisted Search

This tutorial demonstrates how to use experimental powder X-ray diffraction (PXRD) data to guide the crystal structure prediction, combining energy and PXRD similarity in a multi-objective fitness function.

---

## Scenario

- **System**: A rigid organic molecule with known experimental PXRD pattern
- **Z = 4**: 4 molecules per unit cell
- **Backend**: UMA on GPU
- **Fitness**: Combined energy + PXRD similarity (VC-PWDF metric)
- **Requires**: [critic2](https://aoterodelaroza.github.io/critic2/) installed and in PATH

## When to Use PXRD-Assisted Search

PXRD-assisted search is useful when:

- You have an **experimental PXRD pattern** but no solved crystal structure
- You want to **bias the search** toward structures matching the experimental pattern
- You want to identify which predicted structures match the experiment

!!! note "critic2 Required"
    The PXRD similarity calculation uses the **VC-PWDF** (Variable-Cell Powder Difference) metric implemented via [critic2](https://aoterodelaroza.github.io/critic2/). See the [Installation Guide](../getting-started/installation.md#optional-critic2-for-pxrd-fitness) for setup instructions.

## What You Need

| File | Description | Required? |
|------|-------------|-----------|
| `structures.json` | Initial crystal structures (ASE JSON) | Yes |
| `experimental.xy` | Experimental PXRD pattern (2θ vs intensity) | Yes |
| `experimental.cif` | Experimental structure (for removing matches) | Optional |
| `ui.conf` | Configuration file | Yes |
| `submit_gpu.sh` | SLURM submission script | Yes |

## Step 1: Prepare PXRD Data

### Experimental PXRD Pattern

The experimental PXRD pattern should be in a two-column `.xy` format (2θ in degrees, intensity):

```
5.000  12.3
5.010  13.1
5.020  14.8
...
50.000  5.2
```

### Cell Parameters

You also need the **experimental unit cell parameters** for the VC-PWDF calculation:

```
a  b  c  alpha  beta  gamma
```

These are typically available from indexing the PXRD pattern.

## Step 2: Write Configuration

The key difference from a standard energy-only run is using the `vc_energy_fitness` module:

```ini
# ==============================================================================
# GAtor 2.0: PXRD-Assisted CSP with UMA
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
fitness_module = vc_energy_fitness          # <-- Multi-objective fitness

[fitness]
energy_name = energy
pxrd_file = /path/to/experimental.xy       # <-- Experimental PXRD pattern
pxrd_cell = 10.37 12.61 13.85 90 90.27 90  # <-- a b c alpha beta gamma
pxrd_scaling_factor = 0.25                  # <-- Weight of PXRD similarity (0-1)

[initial_pool]
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
stored_vc_name = vc_similarity             # <-- Store PXRD similarity scores
prepare_initial_pool = TRUE
remove_match = TRUE                         # <-- Remove known experimental matches
exp_path = /path/to/experimental.cif        # <-- Experimental structure

[run_settings]
num_molecules = 4
num_atoms = 53
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
target_volume = 1791.81
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

### Key PXRD-Specific Settings

| Setting | Section | Description |
|---------|---------|-------------|
| `fitness_module = vc_energy_fitness` | `[modules]` | Enables multi-objective fitness |
| `pxrd_file` | `[fitness]` | Path to experimental PXRD `.xy` file |
| `pxrd_cell` | `[fitness]` | Experimental cell parameters: `a b c α β γ` |
| `pxrd_scaling_factor` | `[fitness]` | Weight of PXRD similarity (0 = energy only, 1 = PXRD only) |
| `stored_vc_name` | `[initial_pool]` | Property name for storing PXRD similarity scores |
| `remove_match` | `[initial_pool]` | Remove structures matching experimental CIF |
| `exp_path` | `[initial_pool]` | Path to experimental CIF for match removal |

### Choosing `pxrd_scaling_factor`

The combined fitness is:

$$F = (1 - \lambda) \cdot F_{\text{energy}} + \lambda \cdot F_{\text{PXRD}}$$

| Value | Effect |
|-------|--------|
| `0.0` | Pure energy optimization (no PXRD influence) |
| `0.1–0.3` | Mild PXRD bias — energy dominates, PXRD breaks ties |
| `0.25` | **Recommended starting point** |
| `0.5` | Equal weight to energy and PXRD similarity |
| `0.7–1.0` | Strong PXRD bias — useful when the experimental pattern is high-quality |

## Step 3: Submit and Monitor

```bash
sbatch submit_gpu.sh

# Monitor progress
tail -f GAtor.log

# Check energy hierarchy (now includes PXRD similarity)
cat tmp/energy_hierarchy_*.dat
```

## Step 4: Analyze Results

The energy hierarchy file now includes PXRD similarity scores alongside energies. Higher VC-PWDF similarity (closer to 1.0) indicates a better match to the experimental pattern.

```python
import json

# Load pool structures
with open("tmp/pool_0/structure_001.json") as f:
    struct = json.load(f)

# Access PXRD similarity
print(f"Energy: {struct['properties']['energy']}")
print(f"VC similarity: {struct['properties']['vc_similarity']}")
```

---

## Tips

!!! tip "critic2 Verification"
    Before running, verify critic2 is accessible:
    ```bash
    which critic2
    critic2 --help
    ```

!!! tip "PXRD Pattern Quality"
    The quality of PXRD-assisted search depends heavily on the experimental pattern. Ensure your `.xy` file covers sufficient 2θ range (typically 5–50°) and has good signal-to-noise ratio.

!!! tip "Iterative Approach"
    Start with `pxrd_scaling_factor = 0.0` (pure energy) to establish the energy landscape, then increase to 0.25 in a second run to incorporate PXRD guidance.
