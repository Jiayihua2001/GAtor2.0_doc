# Configuration File

GAtor is configured via a single `.conf` file using Python's [ConfigParser](https://docs.python.org/3/library/configparser.html) INI format. Every aspect of the genetic algorithm is controlled through this file.

---

## File Format

```ini
[section_name]
option_name = value
# Comments start with # or ;
list_option = item1 item2 item3
boolean_option = TRUE
numeric_option = 42
```

!!! info "Conventions"
    - **Booleans**: `TRUE` / `FALSE` (case-insensitive)
    - **Lists**: Space-separated values on one line
    - **Paths**: Absolute or relative to the working directory

---

## Section Overview

### `[GAtor_master]` — Master Control

Controls which phases of the GA pipeline to execute.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `fill_initial_pool` | bool | `TRUE` | Fill the initial structure pool |
| `run_ga` | bool | `TRUE` | Run the genetic algorithm |
| `analyze_initial_pool` | bool | `FALSE` | Run pool analysis after filling |
| `check_conf_file` | bool | `TRUE` | Validate configuration before running |
| `testing_mode` | bool | `FALSE` | Enable testing/debugging mode |
| `working_directory` | string | CWD | Override working directory |

### `[modules]` — Module Selection

Select which implementation to use for each GA operator.

| Option | Type | Choices | Description |
|--------|------|---------|-------------|
| `initial_pool_module` | string | `IP_filling` | Initial pool filling strategy |
| `optimization_module` | string | `MACE`, `UMA`, `AIMNET2`, `FHI_aims`, `VASP` | Energy calculator |
| `spe_module` | string | Same as optimization | Single-point energy calculator |
| `comparison_module` | string | `structure_comparison` | Duplicate detection method |
| `selection_module` | string | `tournament_selection`, `roulette_selection`, `Adaptive_tournament_selection`, `AdaptiveMix_selection`, `uniform_selection`, `elite_selection` | Parent selection strategy |
| `mutation_module` | string | `standard_mutation` | Mutation operator |
| `crossover_module` | string | `symmetric_crossover`, `standard_crossover`, `torsion_angle_crossover` | Crossover operator |
| `clustering_module` | string | `cluster` | Clustering algorithm |
| `fitness_module` | string | `standard_energy`, `vc_energy_fitness` | Fitness evaluation |

### `[run_settings]` — GA Parameters

Core settings that define the crystal system and GA behavior.

| Option | Type | Description |
|--------|------|-------------|
| `num_molecules` | int | Number of molecules per unit cell (Z) |
| `Z_prime` | float | Molecules in the asymmetric unit (default: 1) |
| `crystal` | string | Crystal type: `crystal`, `co-crystal` |
| `stoic` | list[int] | Stoichiometry for cocrystals (e.g., `1 1`) |
| `num_atoms` | list[int] | Atoms per molecule type (auto-detected if omitted) |
| `end_ga_structures_added` | int | Stop after this many structures added |
| `output_all_geometries` | bool | Log all intermediate geometries |
| `restart_replicas` | bool | Allow replicas to restart on failure |
| `skip_energy_evaluations` | bool | Skip energy calculations (testing) |
| `optimization_style` | string | `minimize` or `maximize` |
| `flexible` | bool | Enable flexible molecule mode |

### `[initial_pool]` — Initial Pool Settings

| Option | Type | Description |
|--------|------|-------------|
| `user_structures_dir` | string | Directory containing initial structures |
| `stored_energy_name` | string | Energy property name in structures.json |
| `stored_vc_name` | string | Property name for VC fitness score |
| `stored_pwdf_name` | string | Property name for stored PWDF score |
| `prepare_initial_pool` | bool | Auto-build pool from structures.json |
| `reorder_initial_pool` | bool | Reorder atoms to match reference |
| `remove_match` | bool | Remove structures matching experiment |
| `exp_path` | string | Path to experimental CIF for matching |
| `restart` | bool | Resume from save.json checkpoint |

### `[fitness]` — Fitness Function

| Option | Type | Description |
|--------|------|-------------|
| `energy_name` | string | Property name to use as energy fitness |
| `pxrd_file` | string | Path to experimental PXRD pattern (.xy) |
| `pxrd_cell` | string | Cell parameters for PXRD: `a b c alpha beta gamma` |
| `pxrd_scaling_factor` | float | Weight of PXRD in combined fitness |

### `[selection]` — Selection Parameters

| Option | Type | Description |
|--------|------|-------------|
| `fitness_reversal_probability` | float | Probability of reversing fitness ranking (diversity) |
| `tournament_size` | int | Number of candidates per tournament |

### `[crossover]` — Crossover Parameters

| Option | Type | Description |
|--------|------|-------------|
| `crossover_probability` | float | Probability of performing crossover (vs. mutation only) |
| `blend_lat_prob` | float | Probability of blending lattice vectors |
| `blend_lat_std` | float | Lattice blend standard deviation (default: 0.25) |
| `blend_mol_COM_prob` | float | Probability of blending molecular centers of mass |
| `blend_mol_COM_std` | float | COM blend standard deviation (default: 0.25) |
| `swap_mol_geo_prob` | float | Probability of swapping molecular geometries |
| `blend_mol_orien_prob` | float | Probability of blending molecular orientations |
| `blend_mol_orien_std` | float | Orientation blend standard deviation (default: 0.25) |
| `blend_torsion_prob` | float | Probability of blending torsion angles |
| `blend_torsion_std` | float | Torsion blend standard deviation (default: 0.25) |
| `swap_sym_prob` | float | Probability of symmetry swap |

### `[mutation]` — Mutation Parameters

| Option | Type | Description |
|--------|------|-------------|
| `stand_dev_trans` | float | Standard deviation for translation (default: 0.3 A) |
| `stand_dev_rot` | float | Standard deviation for rotation (default: 30 degrees) |
| `stand_dev_strain` | float | Standard deviation for lattice strain (default: 0.3) |
| `double_mutate_prob` | float | Probability of applying a second mutation (default: 0.2) |
| `rotation_method` | string | `gaussian` or `discrete` (default: gaussian) |
| `translation_method` | string | `gaussian` or `discrete` (default: gaussian) |
| `symm_rotation_method` | string | `gaussian` or `discrete` (default: discrete) |
| `symm_translation_method` | string | `gaussian` or `discrete` (default: discrete) |
| `min_torsion_change` | float | Minimum torsion change in degrees (default: 30.0) |
| `torsion_method` | string | `discrete` or `gaussian` (default: discrete) |
| `torsion_prob` | float | Probability of mutating each torsion (default: 0.5) |
| `category_probs` | string | Custom category probability weights |
| `specific_mutations` | list | Restrict to specific mutation types |

Available mutation types:

| Mutation | Category | Description |
|----------|----------|-------------|
| `Rand_trans` | Translation | Random molecular translation |
| `Frame_trans` | Translation | Translation along lattice vectors |
| `Symm_trans` | Translation | Symmetry-preserving translation |
| `Void_aware_translation` | Translation | Translation toward void space |
| `Rand_rot` | Rotation | Random molecular rotation |
| `Frame_rot` | Rotation | Rotation about lattice axes |
| `Symm_rot` | Rotation | Symmetry-preserving rotation |
| `Permutation` | Permutation | Swap molecular positions |
| `Rand_strain` | Strain | Random lattice strain |
| `Sym_strain` | Strain | Symmetry-preserving strain |
| `Vol_strain` | Strain | Volume-changing strain |
| `Angle_strain` | Strain | Angle-changing strain |
| `Rand_block_glide` | Block | Random block glide |
| `Rand_block_permute` | Block | Block permutation |
| `Phonon_mode` | Block | Phonon mode displacement |
| `Herringbone_motif_flip` | Block | Herringbone motif flip |
| `Conformer_mutation` | Conformer | Change molecular conformer |
| `Torsion_angle` | Conformer | Rotate a torsion angle |

### `[cell_check_settings]` — Structure Validation

| Option | Type | Description |
|--------|------|-------------|
| `target_volume` | float | Target unit cell volume (auto-estimated if omitted) |
| `volume_upper_ratio` | float | Max volume ratio vs. target (e.g., 1.4) |
| `volume_lower_ratio` | float | Min volume ratio vs. target (e.g., 0.6) |
| `specific_radius_proportion` | float | SR check: fraction of sum of vdW radii |
| `full_atomic_distance_check` | float | Minimum interatomic distance (Angstroms) |

### `[pre_relaxation_comparison]` — Pre-Relaxation Duplicate Check

| Option | Type | Description |
|--------|------|-------------|
| `ltol` | float | Lattice length tolerance |
| `stol` | float | Site tolerance |
| `angle_tol` | float | Angle tolerance (degrees) |

### `[post_relaxation_comparison]` — Post-Relaxation Duplicate Check

| Option | Type | Description |
|--------|------|-------------|
| `energy_comp_window` | float | Energy window for comparison (eV) |
| `ltol` | float | Lattice length tolerance |
| `stol` | float | Site tolerance |
| `angle_tol` | float | Angle tolerance (degrees) |

### `[clustering]` — Clustering Settings

| Option | Type | Description |
|--------|------|-------------|
| `clustering_algorithm` | string | `AffinityPropagation` |
| `feature_vector` | string | `RSF` or `RCD` |

---

## Full Example

See the [Tutorial: Rigid Molecule with MLIP](../tutorials/rigid-mlip.md) for a complete, annotated configuration file.
