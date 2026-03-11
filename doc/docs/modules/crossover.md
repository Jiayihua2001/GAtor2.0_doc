# Crossover Modules

Crossover operators combine two parent crystal structures to create an offspring that inherits features from both parents.

---

## Module Selection

```ini
[modules]
crossover_module = symmetric_crossover
```

---

## Symmetric Crossover (Recommended)

The default and most versatile crossover operator. Operates on the reduced (symmetry-independent) representation of the crystal and applies multiple operations probabilistically:

```ini
[modules]
crossover_module = symmetric_crossover

[crossover]
crossover_probability = 0.75
blend_lattice_prob = 0.5
blend_COM_prob = 0.5
swap_geo_prob = 0.5
blend_orien_prob = 0.5
blend_torsion_prob = 0.5      # Only with [conformer] section
```

### Operations

Each operation is applied independently with its configured probability:

| Operation | Description |
|-----------|-------------|
| **Blend Lattice** | Interpolate lattice vectors between parents (std = 0.25) |
| **Blend COM** | Interpolate molecular centers of mass (std = 0.25) |
| **Swap Geometry** | Replace molecular geometry from paired parent |
| **Blend Orientation** | Interpolate molecular orientations (std = 0.25) |
| **Blend Torsion** | Interpolate torsion angles, flexible molecules (std = 0.25) |

Each blend operation draws from a Gaussian centered at 0.5 with standard deviation 0.25, producing a weighted interpolation between the two parents.

**Workflow**:

1. Reduce both parents to their asymmetric units
2. Match molecules between parents
3. Apply crossover operations probabilistically
4. Reconstruct the full crystal from the modified asymmetric unit using symmetry operations

!!! info "Symmetry Preservation"
    Symmetric crossover preserves the space group symmetry of the parent structure, producing offspring in the same space group.

---

## Standard Crossover

A simpler crossover that operates on the full unit cell without symmetry reduction:

```ini
[modules]
crossover_module = standard_crossover

[crossover]
crossover_probability = 0.75
```

---

## Torsion Angle Crossover

Specialized for flexible molecules — blends torsion angles between parents when the angular difference exceeds a minimum threshold (20 degrees):

```ini
[modules]
crossover_module = torsion_angle_crossover

[crossover]
crossover_probability = 0.75
blend_torsion_cent = 0.5
blend_torsion_width = 0.3
blend_torsion_cutoff = 0.9
```

---

## Dimer Crossover

Designed for cocrystal systems — swaps molecular orientations in a dimer-based fashion:

```ini
[modules]
crossover_module = crossover_dimers
```

---

## Crossover Probability

The `crossover_probability` controls how often crossover is applied vs. mutation-only:

```ini
[crossover]
crossover_probability = 0.75    # 75% crossover + mutation, 25% mutation only
```

When crossover is skipped, only mutation is applied to a single parent.
