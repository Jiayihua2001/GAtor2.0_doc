# Cocrystals & Z' > 1

GAtor supports crystal structure prediction for multi-component systems (cocrystals) and structures with multiple molecules in the asymmetric unit (Z' > 1).

---

## Cocrystal Setup

A cocrystal contains two or more chemically distinct molecular components in the same crystal lattice (e.g., an API + coformer).

### Configuration

```ini
[run_settings]
crystal = co-crystal
num_molecules = 4          # Total molecules per unit cell
Z_prime = 1                # Molecules per asymmetric unit
stoic = 1 1                # Stoichiometry: 1 of type A, 1 of type B
num_atoms = 20 15          # Atoms in molecule A, atoms in molecule B
```

!!! important "Stoichiometry"
    The `stoic` option defines how many of each molecular type are in the asymmetric unit. For a 1:1 cocrystal with Z = 4, `num_molecules = 4` and `stoic = 1 1` means 2 molecules of type A and 2 of type B per unit cell.

### Molecule Files

For cocrystals, provide separate molecule files:

```
mol_1.in    # Molecule type A
mol_2.in    # Molecule type B
```

These are automatically extracted from the initial pool if `prepare_initial_pool = TRUE`.

### ASU Partner Swap

GAtor automatically performs ASU (Asymmetric Unit) partner swap analysis during initial pool preparation. This assigns each molecule in the structure to the correct component type, ensuring consistent labeling.

---

## Z' > 1 Setup

Z' > 1 means there are multiple symmetry-independent molecules of the same type in the asymmetric unit.

### Configuration

```ini
[run_settings]
crystal = co-crystal       # Uses the multi-component framework
num_molecules = 8           # Total molecules
Z_prime = 2                 # 2 molecules per ASU
stoic = 1                   # Single component
num_atoms = 25              # Atoms per molecule
```

!!! note
    Even for single-component Z' > 1 systems, set `crystal = co-crystal` to enable the multi-molecule machinery.

---

## Crossover for Multi-Component Systems

The `symmetric_crossover` module handles cocrystals by operating on each molecular component independently — blending lattice vectors, centers of mass, and orientations while respecting the stoichiometry.

The `crossover_dimers` module provides an alternative dimer-based crossover specifically designed for cocrystal systems, swapping molecular orientations between parent structures.

---

## Conformer Handling in Cocrystals

For flexible cocrystals, provide separate conformer pools for each component:

```ini
[conformer]
pre_calc_conformer_json_name = conformer_0.json conformer_1.json
```

Where `conformer_0.json` contains conformers for molecule type A and `conformer_1.json` for type B.
