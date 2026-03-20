# Tutorial 3: Cocrystal Prediction

This tutorial demonstrates crystal structure prediction for the **BEDQAG** system — a 1:1 binary cocrystal with two distinct molecular components.

---

## Scenario

- **System**: BEDQAG — 1:1 binary cocrystal (16 + 18 atoms per molecule)
- **Z = 2**: 2 formula units per unit cell → 4 total molecules (2 of type A + 2 of type B)
- **Z' = 1**: One formula unit per asymmetric unit
- **Backend**: UMA on GPU
- **Experimental structure**: CSD entry 2123841

## Files

The ready-to-run example is in `examples/03_cocrystal/BEDQAG/`:

```
03_cocrystal/
└── BEDQAG/
    ├── ui.conf                   # Annotated cocrystal configuration
    ├── structures.json           # Pre-generated initial pool
    ├── 2123841.cif               # Experimental structure
    ├── mol_1.in                  # Molecule type A definition
    └── mol_2.in                  # Molecule type B definition
```

## What Makes Cocrystals Different

| Setting | Single-Component | Cocrystal (1:1) |
|---|---|---|
| `crystal` | `crystal` | `co-crystal` |
| `stoic` | (not set) | `1 1` |
| `num_atoms` | (auto-detected) | `16 18` (per molecule type) |
| `reorder_initial_pool` | `FALSE` | **`TRUE`** (required) |
| `feature_vector` | `RCD` or `RSF` | **`RSF`** (required) |
| Molecule files | `mol.in` (optional) | `mol_1.in` + `mol_2.in` |

### How Z and num_molecules Relate

For a 1:1 cocrystal with Z' = 1:

- **Z** = number of formula units per unit cell (e.g., Z=2)
- **num_molecules** = Z × sum(stoic) = 2 × (1+1) = **4**

## Quick Start

1. **Copy the example**:

    ```bash
    cp -r examples/03_cocrystal/BEDQAG my_cocrystal_run
    cd my_cocrystal_run
    ```

2. **Prepare molecule files** — Create `mol_1.in` and `mol_2.in` in ASE atom format. Count atoms in each for `num_atoms`.

3. **Run pool analysis** — Typical cocrystal SR: 0.75–0.82

    ```bash
    python examples/01_prepare/analyze_pool.py \
        --ui-conf ui.conf --pool-path structures.json \
        --pool-format structures_json --energy-key uma_lbfgs
    ```

4. **Edit `ui.conf`** — Update `exp_path`, `num_atoms`, and `specific_radius_proportion`

5. **Submit**:

    ```bash
    sbatch submit.sh
    ```

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
fitness_module = standard_energy

[fitness]
energy_name = energy

[initial_pool]
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
prepare_initial_pool = TRUE
# Reorder is CRITICAL for cocrystals — ensures consistent atom ordering
reorder_initial_pool = TRUE
remove_match = TRUE
exp_path = 2123841.cif

[run_settings]
# --- Cocrystal-specific settings ---
crystal = co-crystal
num_molecules = 4                    # Z=2, stoic=1 1 → 2×(1+1)=4
Z_prime = 1
stoic = 1 1                          # 1 mol A + 1 mol B per formula unit
num_atoms = 16 18                    # Atoms in mol_1, mol_2 (optional, auto-detected)
end_ga_structures_added = 200
output_all_geometries = TRUE
restart_replicas = TRUE

[parallel_settings]
parallelization_method = srun
run_on_gpu = TRUE
replicas_per_node = 4

[UMA]
store_energy_names = uma uma_lbfgs
# Generous first threshold for cocrystals — relaxation changes energy significantly
relative_energy_thresholds = 100 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500
save_trajectory = FALSE

[selection]
tournament_size = 10

[crossover]
crossover_probability = 0.25         # Lower for cocrystals (more mutations)

[mutation]
stand_dev_trans = 0.3                # Smaller for tighter cocrystal packing
stand_dev_rot = 30
stand_dev_strain = 0.3

[cell_check_settings]
volume_upper_ratio = 1.4
volume_lower_ratio = 0.6
specific_radius_proportion = 0.79    # From pool analysis for BEDQAG

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
# RSF is REQUIRED for cocrystals — RCD does not support multi-component systems
clustering_algorithm = AffinityPropagation
feature_vector = RSF
```

### Key Cocrystal-Specific Settings

| Setting | Value | Why |
|---|---|---|
| `crystal = co-crystal` | Enables cocrystal mode | Stoichiometry-preserving operators |
| `stoic = 1 1` | 1:1 stoichiometry | Auto-detected from `mol_1.in` / `mol_2.in` |
| `num_atoms = 16 18` | Atoms per molecule type | Auto-detected if not set |
| `reorder_initial_pool = TRUE` | Required | Ensures consistent atom grouping |
| `feature_vector = RSF` | Required | RCD does not support multi-component |

### Crossover Options

GAtor provides two crossover modes for cocrystals:

=== "Symmetric Crossover (Default)"

    Handles each molecular type independently — lattice, COM positions, and orientations are crossed over per component type.

    ```ini
    [modules]
    crossover_module = symmetric_crossover
    ```

=== "Dimer Crossover"

    Treats molecular dimers (A-B pairs) as single units. Useful when the A-B intermolecular interaction is the dominant packing force.

    ```ini
    [modules]
    crossover_module = crossover_dimers
    ```

## Z' > 1 (Single Component)

For single-component systems with Z' > 1 (multiple symmetry-independent molecules), use cocrystal mode with a single-type stoichiometry:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 8       # Z'=2 with Z=4 → 8 molecules
Z_prime = 2
stoic = 1               # Single component
num_atoms = 30           # Atoms per molecule
```

This allows GAtor to treat each symmetry-independent molecule separately during crossover and mutation.

---

## Tips

!!! tip "Volume Estimation"
    For cocrystals, the target volume accounts for both molecular types. If you omit `target_volume`, GAtor estimates it from the initial structures. For manual estimation: `target_volume ≈ (V_mol_A + V_mol_B) × Z / stoic_sum`.

!!! tip "Rejection Rate"
    If the GA rejects too many structures, lower `specific_radius_proportion` or widen `volume_upper_ratio` / `volume_lower_ratio`.

!!! tip "Stoichiometry"
    The `stoic` setting defines the ratio, not absolute counts. For a 2:1 cocrystal (2 API + 1 coformer), use `stoic = 2 1` with `num_molecules = 6` (for Z'=1, Z=2).

!!! tip "Restart"
    If the job times out, resubmit — GAtor resumes from the existing pool and maintains all cocrystal metadata.

---

## Next Steps

- [Tutorial 4: Flexible Molecule](flexible.md) — Conformational flexibility
- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
