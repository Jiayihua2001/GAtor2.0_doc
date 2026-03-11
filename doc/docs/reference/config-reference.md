# Configuration Reference

Complete reference of all configuration sections and options in `ui.conf`.

---

## `[GAtor_master]`

Master control switches for the GA pipeline.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `fill_initial_pool` | bool | `TRUE` | Run initial pool filling phase |
| `run_ga` | bool | `TRUE` | Run the genetic algorithm |
| `analyze_initial_pool` | bool | `FALSE` | Run post-fill pool analysis |
| `check_conf_file` | bool | `TRUE` | Validate config before execution |
| `testing_mode` | bool | `FALSE` | Enable test/debug mode |
| `working_directory` | string | CWD | Override working directory path |

---

## `[modules]`

Select module implementations for each GA component.

| Option | Type | Choices |
|--------|------|---------|
| `initial_pool_module` | string | `IP_filling` |
| `optimization_module` | string | `MACE`, `UMA`, `AIMNET2`, `FHI_aims`, `VASP` |
| `spe_module` | string | Same as `optimization_module` |
| `comparison_module` | string | `structure_comparison` |
| `selection_module` | string | `tournament_selection`, `roulette_selection`, `Adaptive_tournament_selection`, `AdaptiveMix_selection`, `uniform_selection`, `elite_selection` |
| `mutation_module` | string | `standard_mutation` |
| `crossover_module` | string | `symmetric_crossover`, `standard_crossover`, `torsion_angle_crossover`, `crossover_dimers` |
| `clustering_module` | string | `cluster` |
| `fitness_module` | string | `standard_energy`, `vc_energy_fitness` |

---

## `[run_settings]`

Core GA and crystal system parameters.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `num_molecules` | int | *required* | Total molecules per unit cell (Z) |
| `Z_prime` | float | `1` | Molecules in asymmetric unit |
| `crystal` | string | `crystal` | Crystal type: `crystal` or `co-crystal` |
| `stoic` | list[int] | `None` | Stoichiometry for co-crystals |
| `num_atoms` | list[int] | *auto* | Atoms per molecule type |
| `end_ga_structures_added` | int | *required* | GA termination criterion |
| `output_all_geometries` | bool | `FALSE` | Log all intermediate geometries |
| `restart_replicas` | bool | `FALSE` | Allow replica restart on failure |
| `skip_energy_evaluations` | bool | `FALSE` | Skip energy calculations |
| `optimization_style` | string | `minimize` | `minimize` or `maximize` |
| `flexible` | bool | `FALSE` | Enable flexible molecule mode |

---

## `[initial_pool]`

Initial structure pool configuration.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `user_structures_dir` | string | *required* | Directory for pool structures |
| `stored_energy_name` | string | `energy` | Energy property name in input |
| `stored_vc_name` | string | `None` | VC fitness property name |
| `stored_pwdf_name` | string | `None` | PWDF property name |
| `prepare_initial_pool` | bool | `FALSE` | Auto-build from structures.json |
| `reorder_initial_pool` | bool | `FALSE` | Reorder atoms to match reference |
| `remove_match` | bool | `FALSE` | Remove experimental matches |
| `exp_path` | string | `None` | Path to experimental CIF |
| `restart` | bool | `FALSE` | Resume from save.json |

---

## `[fitness]`

Fitness function configuration.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `energy_name` | string | `energy` | Property name for energy fitness |
| `pxrd_file` | string | `None` | Experimental PXRD file path |
| `pxrd_cell` | string | `None` | Cell params: `a b c alpha beta gamma` |
| `pxrd_scaling_factor` | float | `0.0` | PXRD weight in combined fitness |

---

## `[parallel_settings]`

Parallelization and resource allocation.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `parallelization_method` | string | *required* | `srun`, `mpirun`, or `serial` |
| `run_on_gpu` | bool | `FALSE` | Enable GPU acceleration |
| `replicas_per_node` | int | `1` | GA replicas per compute node |
| `processes_per_replica` | int | `1` | CPU processes per replica |
| `cpus_per_replica` | int | `1` | CPUs allocated per replica |
| `nodes_per_replica` | float | `1` | Nodes per replica (can be < 1) |
| `number_of_replicas` | int | auto | Override total replica count |
| `python_command` | string | `python` | Python executable |
| `aims_processes_per_replica` | int | `1` | MPI ranks for FHI-aims |
| `srun_memory_per_core` | int | `None` | Memory per core (MB) for srun |
| `srun_gator_memory` | int | `None` | Memory for GAtor process (MB) |

---

## `[selection]`

Selection operator parameters.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `tournament_size` | int | *required for tournament* | Tournament candidates |
| `fitness_reversal_probability` | float | `0.0` | Probability of reversed ranking |

---

## `[crossover]`

Crossover operator parameters.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `crossover_probability` | float | `0.75` | Probability of crossover vs mutation-only |
| `blend_lat_prob` | float | `0.5` | Lattice blending probability |
| `blend_lat_std` | float | `0.25` | Lattice blend standard deviation |
| `blend_lat_cent` | float | `0.5` | Lattice blend center |
| `blend_mol_COM_prob` | float | `0.5` | COM blending probability |
| `blend_mol_COM_std` | float | `0.25` | COM blend standard deviation |
| `blend_mol_COM_cent` | float | `0.5` | COM blend center |
| `swap_mol_geo_prob` | float | `0.5` | Geometry swap probability |
| `blend_mol_orien_prob` | float | `0.5` | Orientation blending probability |
| `blend_mol_orien_std` | float | `0.25` | Orientation blend standard deviation |
| `blend_mol_orien_cent` | float | `0.5` | Orientation blend center |
| `blend_torsion_prob` | float | `0.5` | Torsion blending probability |
| `blend_torsion_std` | float | `0.25` | Torsion blend standard deviation |
| `blend_torsion_cent` | float | `0.5` | Torsion blend center |
| `swap_sym_prob` | float | `0.5` | Symmetry swap probability |

---

## `[mutation]`

Mutation operator parameters.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `stand_dev_trans` | float | `0.3` | Translation std dev (Angstroms) |
| `stand_dev_rot` | float | `30` | Rotation std dev (degrees) |
| `stand_dev_strain` | float | `0.3` | Strain std dev (fractional) |
| `double_mutate_prob` | float | `0.2` | Probability of applying a second mutation |
| `rotation_method` | string | `gaussian` | `gaussian` or `discrete` |
| `translation_method` | string | `gaussian` | `gaussian` or `discrete` |
| `symm_rotation_method` | string | `discrete` | `gaussian` or `discrete` |
| `symm_translation_method` | string | `discrete` | `gaussian` or `discrete` |
| `min_torsion_change` | float | `30.0` | Minimum torsion angle change (degrees) |
| `torsion_method` | string | `discrete` | `discrete` or `gaussian` |
| `torsion_prob` | float | `0.5` | Probability of mutating each torsion |
| `category_probs` | string | auto | Category probability overrides |
| `specific_mutations` | list | all | Restrict to listed mutations |

---

## `[cell_check_settings]`

Structure validation parameters.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `target_volume` | float | *auto* | Target unit cell volume (A^3) |
| `volume_upper_ratio` | float | `1.4` | Max volume / target ratio |
| `volume_lower_ratio` | float | `0.6` | Min volume / target ratio |
| `specific_radius_proportion` | float | `0.65` | SR check fraction |
| `full_atomic_distance_check` | float | `0.1` | Min interatomic distance (A) |

---

## `[pre_relaxation_comparison]`

Pre-relaxation duplicate detection.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `ltol` | float | `0.5` | Lattice length tolerance |
| `stol` | float | `0.5` | Site tolerance |
| `angle_tol` | float | `10` | Angle tolerance (degrees) |

---

## `[post_relaxation_comparison]`

Post-relaxation duplicate detection.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `energy_comp_window` | float | `1.5` | Energy window (eV) |
| `ltol` | float | `0.5` | Lattice length tolerance |
| `stol` | float | `0.5` | Site tolerance |
| `angle_tol` | float | `10` | Angle tolerance (degrees) |

---

## `[clustering]`

Clustering configuration.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `clustering_algorithm` | string | `AffinityPropagation` | Algorithm name |
| `feature_vector` | string | `RSF` | `RSF` or `RCD` |

---

## `[conformer]`

Conformer handling (flexible molecules).

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `pre_calc_conformer_json_name` | list | `conformer.json` | Conformer pool file(s) |
| `rmsd_threshold` | float | `1.5` | RMSD threshold for conformer matching (A) |
| `conformer_selection_type` | string | `boltzmann_fitness` | How to select conformers |
| `boltzmann_scale_factor` | float | `5.0` | Boltzmann temperature scaling factor |
| `volume_mult` | float | `1.2` | Volume multiplier for conformer structures |
| `add_conformer_only_if_better` | bool | `TRUE` | Only add conformer if energy improves |
| `add_conformer_energy_quantile` | float | `0.5` | Energy quantile threshold for adding conformers |

---

## `[pool_analysis]`

Initial pool analysis settings (optional).

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `sr_min` | float | `0.60` | Min SR value for analysis |
| `sr_max` | float | `1.05` | Max SR value for analysis |
| `sr_step` | float | `0.01` | SR step size |
| `exp_file` | string | `None` | Experimental CIF for comparison |

---

## Optimization Backend Sections

### `[MACE]`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `store_energy_names` | list | *required* | Energy property names per step |
| `relative_energy_thresholds` | list[float] | `None` | Relative energy cutoffs (eV) |
| `absolute_energy_thresholds` | list[float] | `None` | Absolute energy cutoffs (eV) |
| `reject_if_worst_energy` | list[bool] | `FALSE` | Reject worst-energy structures |
| `fmax` | float | `0.01` | Force convergence (eV/A) |
| `steps` | int | `1000` | Max optimization steps |
| `save_trajectory` | bool | `FALSE` | Save relaxation trajectory |

### `[UMA]`

Same options as `[MACE]`, plus:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `head` | string | `omc` | UMA model head |

### `[AIMNET2]`

Same options as `[MACE]`, plus:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `checkpoint_path` | string | *required* | Path to model checkpoint |

### `[FHI-aims]`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `execute_command` | string | *required* | `srun` or `mpirun` |
| `path_to_aims_executable` | string | *required* | Path to aims binary |
| `control_in_directory` | string | *required* | Control files directory |
| `control_in_filelist` | list | *required* | Control file names |
| `store_energy_names` | list | *required* | Energy property names |
| `relative_energy_thresholds` | list[float] | `None` | Energy cutoffs per step |
| `reject_if_worst_energy` | list[bool] | `FALSE` | Reject worst energy |
| `save_failed_calc` | bool | `TRUE` | Save failed calculations |
| `save_successful_calc` | bool | `TRUE` | Save successful calculations |
| `save_aims_output` | bool | `TRUE` | Keep aims output files |
| `monitor_execution` | bool | `FALSE` | Monitor job progress |
| `update_poll_times` | list[int] | `1200` | Timeout per step (seconds) |

### `[VASP]`

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `path_to_vasp_executable` | string | *required* | Path to VASP binary |
| `store_energy_names` | list | *required* | Energy property names |
| `relative_energy_thresholds` | list[float] | `None` | Energy cutoffs |
| `reject_if_worst_energy` | list[bool] | `FALSE` | Reject worst energy |
