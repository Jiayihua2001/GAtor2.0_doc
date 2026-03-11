# Tutorial 2: Cocrystal Prediction

This tutorial demonstrates crystal structure prediction for a binary cocrystal (two distinct molecular components).

---

## Scenario

- **System**: A 1:1 cocrystal (API + coformer)
- **Z = 4**: 2 molecules of type A + 2 molecules of type B per unit cell
- **Backend**: UMA on GPU

## Configuration

Key differences from a single-component run:

```ini
[run_settings]
crystal = co-crystal
num_molecules = 4
Z_prime = 1
stoic = 1 1                # 1:1 stoichiometry
num_atoms = 20 15           # 20 atoms in mol A, 15 in mol B

[initial_pool]
prepare_initial_pool = TRUE
stored_energy_name = uma_lbfgs
user_structures_dir = initial_pool
```

## Step 1: Prepare Multi-Component Structures

Your `structures.json` must contain cocrystal structures with both molecular types. The molecules should be ordered consistently: all atoms of type A first, then type B, repeated Z times.

## Step 2: Configure Crossover

For cocrystals, `symmetric_crossover` handles each molecular type independently:

```ini
[modules]
crossover_module = symmetric_crossover

[crossover]
crossover_probability = 0.75
```

Alternatively, use the dimer-based crossover:

```ini
[modules]
crossover_module = crossover_dimers
```

## Step 3: Run

The workflow is identical to single-component runs. GAtor automatically:

- Extracts `mol_1.in` and `mol_2.in` from the initial pool
- Performs ASU partner swap to ensure consistent molecule labeling
- Handles stoichiometry during crossover and mutation

## Output

The energy hierarchy and pool files contain the same information as single-component runs, with additional properties tracking each molecular component.

---

## Tips

!!! tip "Atom Ordering"
    Ensure atoms are ordered consistently across all structures. Use `reorder_initial_pool = TRUE` if needed.

!!! tip "Z' > 1"
    For Z' > 1 single-component systems, use `crystal = co-crystal` with `stoic = 1` and the appropriate `Z_prime` value.
