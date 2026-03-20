# Tutorial 3: Cocrystal Prediction

Predict the crystal structure of **BEDQAG** — a 1:1 binary cocrystal with two distinct molecular components (16 + 18 atoms per molecule, Z=2).

!!! info "Prerequisites"
    Read [Tutorial 2](rigid-mlip.md) first — this tutorial shows only the settings that differ from the baseline configuration.

---

## Quick Start

The ready-to-run example is in `examples/03_cocrystal/BEDQAG/`:

1. **Copy the example**:

    ```bash
    cp -r examples/03_cocrystal/BEDQAG my_cocrystal_run
    cd my_cocrystal_run
    ```

2. **Edit `ui.conf`** — update `exp_path` and verify `num_atoms` matches your molecule files (`mol_1.in`, `mol_2.in`)

3. **Edit `submit.sh`** — set your allocation and activate the environment

4. **Submit**: `sbatch submit.sh`

---

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

- **Z** = formula units per unit cell (e.g., Z=2)
- **num_molecules** = Z × sum(stoic) = 2 × (1+1) = **4**

---

## Configuration Changes

Starting from the [baseline config](rigid-mlip.md#configuration-reference), change these sections:

```ini
[initial_pool]
# ... (same as baseline, plus:)
reorder_initial_pool = TRUE                  # Required — ensures consistent atom grouping

[run_settings]
crystal = co-crystal                         # Enable cocrystal mode
num_molecules = 4                            # Z=2, stoic=1 1 → 2×(1+1)=4
Z_prime = 1
stoic = 1 1                                  # 1 mol A + 1 mol B per formula unit
num_atoms = 16 18                            # Atoms in mol_1, mol_2
end_ga_structures_added = 200

[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 100 3           # Tighter first threshold for cocrystals
reject_if_worst_energy = TRUE TRUE

[cell_check_settings]
specific_radius_proportion = 0.79            # From pool analysis for BEDQAG

[clustering]
clustering_algorithm = AffinityPropagation
feature_vector = RSF                         # Required — RCD does not support multi-component
```

All other sections (`[GAtor_master]`, `[modules]`, `[fitness]`, `[parallel_settings]`, `[selection]`, `[crossover]`, `[mutation]`, comparisons) remain the same as the baseline.

### Crossover Options

=== "Symmetric Crossover (Default)"

    Crosses over each molecular type independently — lattice, COM positions, and orientations per component.

    ```ini
    crossover_module = symmetric_crossover
    ```

=== "Dimer Crossover"

    Treats A-B pairs as single units. Useful when the A-B intermolecular interaction is the dominant packing force.

    ```ini
    crossover_module = crossover_dimers
    ```

---

## Z' > 1 (Single Component)

For single-component systems with multiple symmetry-independent molecules:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 8       # Z'=2 with Z=4 → 8 molecules
Z_prime = 2
stoic = 1               # Single component
num_atoms = 30           # Atoms per molecule
```

---

## Tips

!!! tip "Stoichiometry"
    `stoic` defines the ratio, not absolute counts. For a 2:1 cocrystal, use `stoic = 2 1` with `num_molecules = Z × (2+1)`.

!!! tip "Rejection Rate"
    If the GA rejects too many structures, lower `specific_radius_proportion` or widen `volume_upper_ratio` / `volume_lower_ratio`.

!!! tip "Restart"
    If the job times out, resubmit — GAtor resumes from the existing pool.

---

## Next Steps

- [Tutorial 4: Flexible Molecule](flexible.md) — Conformational flexibility
- [Tutorial 5: PXRD-Assisted Search](pxrd-assisted.md) — Add PXRD guidance
- [Tutorial 6: Post-Analysis](post-analysis.md) — Analyze results
