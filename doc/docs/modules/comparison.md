# Comparison Module

The comparison module detects duplicate structures to maintain diversity in the pool. New structures are compared against existing pool members before being added.

---

## Configuration

```ini
[modules]
comparison_module = structure_comparison

[pre_relaxation_comparison]
ltol = 0.5
stol = 0.5
angle_tol = 10

[post_relaxation_comparison]
energy_comp_window = 1.5
ltol = 0.5
stol = 0.5
angle_tol = 10
```

---

## Two-Stage Comparison

GAtor performs duplicate checking at two stages:

### Pre-Relaxation Comparison

Before energy evaluation, the raw offspring is compared against the pool. This avoids wasting computational resources on structures that are already represented.

| Option | Type | Description |
|--------|------|-------------|
| `ltol` | float | Fractional length tolerance (lattice vectors) |
| `stol` | float | Site tolerance (fractional coordinates) |
| `angle_tol` | float | Angle tolerance in degrees |

### Post-Relaxation Comparison

After energy evaluation, the relaxed structure is compared again. This catches cases where different initial structures relax to the same minimum.

| Option | Type | Description |
|--------|------|-------------|
| `energy_comp_window` | float | Only compare structures within this energy window (eV) |
| `ltol` | float | Fractional length tolerance |
| `stol` | float | Site tolerance |
| `angle_tol` | float | Angle tolerance in degrees |

!!! tip "Energy Window"
    The `energy_comp_window` restricts comparison to structures with similar energies, greatly reducing the computational cost of duplicate checking in large pools.

---

## Qualifying Duplicate Saving

When a structure is detected as a duplicate, GAtor can automatically save it to a separate **duplicates pool** if it meets certain criteria. This feature captures structurally similar but not identical structures (e.g., different local minima of the same basin) that may be valuable for analysis.

A duplicate is saved only when **both** conditions are met:

1. **RMSD threshold**: The RMSD between the new structure and its match is **>= 0.2 A** (avoids saving true duplicates)
2. **Energy criterion** (depends on comparison stage):
    - **Pre-relaxation**: The matched structure is the current **lowest-energy** structure in the pool
    - **Post-relaxation**: The new structure has the **global lowest energy**

Saved duplicates are written to the `duplicates/` pool directory and tracked in a `GA_duplicates.dat` log file:

```
# GA_duplicates.dat
new_struct_id    matched_struct_id    rmsd    comparison_stage
a1b2c3d4         e5f6g7h8             0.35    post_relaxation
```

!!! info "Use Case"
    Qualifying duplicate saving is particularly useful for identifying polymorphic structures that relax to similar but distinct minima. These near-duplicates can reveal important information about the energy landscape topology.

---

## Under the Hood

The comparison uses pymatgen's `StructureMatcher`, which accounts for:

- Lattice transformations
- Atom permutations
- Periodic boundary conditions
- Partial occupancy
