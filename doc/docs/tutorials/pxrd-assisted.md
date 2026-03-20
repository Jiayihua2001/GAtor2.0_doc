# Tutorial 5: PXRD-Assisted Search

This tutorial demonstrates how to use experimental powder X-ray diffraction (PXRD) data to guide the crystal structure prediction, combining energy and PXRD similarity in a multi-objective fitness function. It also covers **post-GA fine-tuning** of candidate structures.

---

## Scenario

- **System**: Uracil — rigid molecule with known experimental PXRD pattern
- **Z = 4**: 4 molecules per unit cell
- **Backend**: UMA on GPU
- **Fitness**: Combined energy + PXRD similarity (VC-PWDF metric)
- **Requires**: [critic2](https://aoterodelaroza.github.io/critic2/) installed and in PATH

## Files

```
05_pxrd_assisted/
├── Uracil/                       # PXRD-assisted GA run
│   ├── ui.conf                   # PXRD-guided configuration
│   ├── structures.json           # Initial pool
│   ├── uracil-lqlt.xy           # Experimental PXRD pattern
│   └── 1278441.cif              # Experimental structure
└── fine-tune/                    # Post-GA PXRD refinement
    ├── README.md
    ├── ui.conf                   # Fine-tune configuration
    ├── uracil-lqlt.xy           # Same experimental pattern
    └── initial_pool/             # Candidate structures from GA
```

## When to Use PXRD-Assisted Search

- You have an **experimental PXRD pattern** but no solved crystal structure
- You want to **bias the search** toward structures matching the experimental pattern
- You want to identify which predicted structures match the experiment

!!! note "Prerequisites"
    1. **critic2** installed and on PATH — verify with `which critic2`
    2. Experimental PXRD pattern (`.xy` file: two-column, 2θ vs intensity)
    3. Experimental unit cell parameters (a, b, c, α, β, γ)

## How It Works

The fitness function blends energy and PXRD similarity:

$$F = (1 - \lambda) \cdot F_{\text{energy}} + \lambda \cdot F_{\text{PXRD}}$$

where λ is the `pxrd_scaling_factor`.

## PXRD File Format

Two-column `.xy` file (no header, whitespace-separated):

```
5.000  12.3
5.010  13.1
5.020  14.8
...
40.000  5.2
```

Column 1: 2θ (degrees). Column 2: intensity (arbitrary units, normalized internally).

## Choosing `pxrd_scaling_factor`

| Value | Behavior | Use Case |
|---|---|---|
| `0.00` | Pure energy | Baseline comparison |
| `0.10–0.25` | Gentle PXRD guidance | Noisy or low-resolution data |
| `0.25–0.50` | Balanced | General-purpose |
| `0.50–0.75` | PXRD-dominant | High-quality data |
| **`0.90`** | **Strong PXRD bias (recommended)** | **Production runs** |
| `1.00` | Pure PXRD | Not recommended (no energy regularization) |

## Configuration Walkthrough

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
# vc_energy_fitness enables PXRD-guided multi-objective fitness
fitness_module = vc_energy_fitness

[fitness]
energy_name = energy
# Path to experimental PXRD pattern
pxrd_file = uracil-lqlt.xy
# Experimental unit cell: a b c alpha beta gamma
pxrd_cell = 3.6691 10.3146 12.3958 90 90 96.637
# Weight of PXRD similarity in fitness (0.0–1.0)
pxrd_scaling_factor = 0.90

[initial_pool]
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
stored_pwdf_name = vc_similarity
prepare_initial_pool = TRUE
remove_match = TRUE
exp_path = ./1278441.cif

[run_settings]
num_molecules = 4
end_ga_structures_added = 200
output_all_geometries = TRUE

[parallel_settings]
parallelization_method = srun
replicas_per_node = 4
run_on_gpu = TRUE

[UMA]
store_energy_names = uma uma_lbfgs
# Both thresholds generous — fitness depends on PXRD match, not just energy
relative_energy_thresholds = 1000 1000
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 1500
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.75

[mutation]
stand_dev_trans = 0.3
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.79

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
|---|---|---|
| `fitness_module = vc_energy_fitness` | `[modules]` | Enables multi-objective fitness |
| `pxrd_file` | `[fitness]` | Path to experimental PXRD `.xy` file |
| `pxrd_cell` | `[fitness]` | Experimental cell parameters: `a b c α β γ` |
| `pxrd_scaling_factor` | `[fitness]` | Weight of PXRD similarity (0–1) |
| `stored_pwdf_name` | `[initial_pool]` | Property name for PXRD similarity scores |

!!! important "Energy Thresholds for PXRD Mode"
    Both `relative_energy_thresholds` are set to `1000` because the fitness function depends on PXRD similarity, not just energy. Tight thresholds would reject structures that match the experimental PXRD but are not the lowest in energy.

## Combining with Other Crystal Types

PXRD guidance is an overlay — add it to any crystal type:

| Crystal Type | Additional Settings |
|---|---|
| Rigid | None (default) |
| Flexible | `flexible = TRUE` + `[conformer]` section |
| Cocrystal | `crystal = co-crystal` + stoichiometry settings |

---

## Post-GA Fine-Tuning

<p align="center">
  <img src="../assets/images/fine_tune.png" alt="Fine-Tune Workflow" width="400">
</p>

After the GA completes, you can refine the best candidates against the experimental PXRD pattern using **local mutations**. This is done with the fine-tune mode.

### How Fine-Tuning Works

1. Load candidate structures from a completed GA run
2. Calculate VC-PWDF score for each (lower = better match)
3. For each generation:
    - Select the best-matching structure (greedy)
    - Generate N small mutations (translations, rotations, strains)
    - Evaluate VC-PWDF in parallel (multiprocessing)
    - Keep improvements, replace duplicates if better
4. Output structures ranked by PXRD similarity

### Fine-Tune Configuration

```ini
[GAtor_master]
fill_initial_pool = FALSE
run_ga = FALSE
fine_tune = TRUE                     # Enable fine-tune mode

[modules]
optimization_module = UMA
# Other modules remain the same

[fitness]
energy_name = energy
pxrd_file = uracil-lqlt.xy
pxrd_cell = 3.6691 10.3146 12.3958 90 90 96.637

[initial_pool]
user_structures_dir = ./initial_pool
stored_energy_name = uma_lbfgs
stored_vc_name = vc_similarity
prepare_initial_pool = FALSE

[run_settings]
crystal = crystal
num_molecules = 4
end_ga_structures_added = 100
output_all_geometries = TRUE

[parallel_settings]
parallelization_method = serial

[fine_tune]
# CPU processes for parallel PXRD calculations
num_processes = 128
# Mutation trials per generation
num_trial = 100
# PXRD evaluations per mutated structure
num_ga_structures = 20
# Maximum generations before stopping
max_generations = 1000
# Stop after N generations without improvement
max_no_improvement = 20
# Duplicate detection (tight tolerances for fine-tuning)
tight_ltol = 0.1
tight_stol = 0.15
tight_angle_tol = 3.0
# Convergence
vc_improvement_threshold = 0.01
max_recursive_iterations = 5

[mutation]
# Small perturbations for local refinement
stand_dev_trans = 0.05
stand_dev_rot = 5
stand_dev_strain = 0.05
stand_dev_cell_angle = 2.0
torsion_sigma = 5.0
specific_mutations = Rand_trans Rand_rot Frame_trans Frame_rot Rand_strain Sym_strain Vol_strain Angle_strain Symm_rot Symm_trans Torsion_angle Phonon_mode
```

### Fine-Tune Key Parameters

| Parameter | Default | Description |
|---|---|---|
| `num_processes` | 4 | CPU processes for parallel PXRD evaluation |
| `num_trial` | 20 | Mutation trials per generation |
| `max_generations` | 100 | Maximum fine-tune generations |
| `max_no_improvement` | 10 | Stop after N generations without improvement |
| `stand_dev_trans` | 0.05 | Translation perturbation (much smaller than GA) |
| `stand_dev_rot` | 5 | Rotation perturbation (much smaller than GA) |

### Running the Fine-Tune

```bash
# Prepare candidate structures from GA output
python examples/01_prepare/prepare_structures.py /path/to/ga_output_cifs \
    --output structures.json

# Run fine-tuning (CPU, no GPU needed)
run_gator ui.conf
```

### Fine-Tune Output

Structures are named with their PWDF score:

```
best_pwdf_0.031637_struct_04dbd973f5.cif
best_pwdf_0.045123_struct_a1b2c3d4e5.cif
```

Lower PWDF = better match to the experimental pattern. Values below **0.05** indicate very good agreement.

---

## Tips

!!! tip "critic2 Verification"
    Before running, verify critic2 is accessible:
    ```bash
    which critic2
    critic2 --help
    ```

!!! tip "Thread Control"
    Set `OMP_NUM_THREADS=1` to avoid thread conflicts between MLIP and critic2.

!!! tip "PXRD Pattern Quality"
    Ensure your `.xy` file covers sufficient 2θ range (typically 5–50°) and has good signal-to-noise ratio.

!!! tip "Two-Stage Workflow"
    For best results, run energy-only GA first to establish the energy landscape, then PXRD-assisted for refinement. Use the fine-tune mode as a final local optimization step.

!!! tip "Fine-Tune Runtime"
    Runtime is dominated by critic2 (PXRD calculation), not structure manipulation. More `num_processes` = faster, but watch memory.

---

## Next Steps

- [Tutorial 6: Post-Analysis](post-analysis.md) — Convergence plots and structure extraction
- [Tutorial 7: CSP Landscape Viewer](csp-viewer.md) — Interactive analysis
