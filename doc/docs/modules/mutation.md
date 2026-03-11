# Mutation Modules

Mutation operators apply random perturbations to crystal structures, introducing new structural diversity that crossover alone cannot achieve.

---

## Module Selection

```ini
[modules]
mutation_module = standard_mutation

[mutation]
stand_dev_trans = 0.3
stand_dev_rot = 30
stand_dev_strain = 0.3
double_mutate_prob = 0.2
```

---

## Hierarchical Mutation Selection

GAtor uses a **two-level hierarchical** mutation selection system:

1. First, a **mutation category** is chosen with equal probability
2. Then, a **specific method** within that category is chosen uniformly

This ensures that each *type* of perturbation (translation, rotation, strain, etc.) has an equal chance of being applied, regardless of how many methods exist in each category.

### Categories and Methods

| Category | Methods | Description |
|----------|---------|-------------|
| **Translation** | `Rand_trans`, `Frame_trans`, `Symm_trans`, `Void_aware_translation` | Move molecules |
| **Rotation** | `Rand_rot`, `Frame_rot`, `Symm_rot` | Rotate molecules |
| **Permutation** | `Permutation` | Swap molecular positions |
| **Strain** | `Rand_strain`, `Sym_strain`, `Vol_strain`, `Angle_strain` | Deform the lattice |
| **Block** | `Rand_block_glide`, `Rand_block_permute`, `Phonon_mode`, `Herringbone_motif_flip` | Layer/block operations |
| **Conformer** | `Conformer_mutation`, `Torsion_angle` | Change molecular shape |

---

## Translation Mutations

| Method | Description |
|--------|-------------|
| `Rand_trans` | Random displacement with Gaussian distribution (`stand_dev_trans`) |
| `Frame_trans` | Translation along lattice vector directions |
| `Symm_trans` | Translation preserving space group symmetry (discrete by default) |
| `Void_aware_translation` | Translate molecules toward regions of low density |

```ini
[mutation]
stand_dev_trans = 0.3          # Standard deviation in Angstroms
translation_method = gaussian  # gaussian or discrete
symm_translation_method = discrete
```

## Rotation Mutations

| Method | Description |
|--------|-------------|
| `Rand_rot` | Random rotation with Gaussian distribution (`stand_dev_rot`) |
| `Frame_rot` | Rotation about lattice vector axes |
| `Symm_rot` | Rotation preserving space group symmetry (discrete by default) |

```ini
[mutation]
stand_dev_rot = 30             # Standard deviation in degrees
rotation_method = gaussian     # gaussian or discrete
symm_rotation_method = discrete
```

## Strain Mutations

| Method | Description |
|--------|-------------|
| `Rand_strain` | Random symmetric strain tensor |
| `Sym_strain` | Symmetry-preserving strain |
| `Vol_strain` | Volume-changing isotropic strain |
| `Angle_strain` | Modify cell angles only |

```ini
[mutation]
stand_dev_strain = 0.3    # Standard deviation (fractional)
```

## Block Mutations

| Method | Description |
|--------|-------------|
| `Rand_block_glide` | Glide a block of molecules along a crystal plane |
| `Rand_block_permute` | Permute molecules within a block |
| `Phonon_mode` | Displace molecules along a phonon mode |
| `Herringbone_motif_flip` | Flip the herringbone packing motif |

## Conformer Mutations

Only available when the `[conformer]` section is configured:

| Method | Description |
|--------|-------------|
| `Conformer_mutation` | Replace molecule with a different conformer |
| `Torsion_angle` | Randomly rotate a torsion angle |

---

## Double Mutation

GAtor can apply two mutations sequentially to a single structure:

```ini
[mutation]
double_mutate_prob = 0.2    # 20% chance of applying a second mutation
```

---

## Torsion Angle Mutation

For flexible molecules, torsion angle mutations rotate individual dihedral angles:

```ini
[mutation]
min_torsion_change = 30.0    # Minimum torsion angle change (degrees)
torsion_method = discrete    # discrete or gaussian
torsion_prob = 0.5           # Probability of mutating each torsion
```

---

## Restricting Mutations

To use only specific mutation types:

```ini
[mutation]
specific_mutations = Rand_trans Rand_rot Rand_strain Conformer_mutation
```

If not specified, all applicable mutations are used.

---

## Category Probabilities

The hierarchical mutation selection can be customized with explicit category probabilities:

```ini
[mutation]
# Default when flexible=FALSE: 3 equal classes (Translation&Rotation, Permutation&Block, Strain)
# Default when flexible=TRUE:  Translation&Rotation:0.3 Permutation&Block:0.3 Strain:0.3 Conformer:0.1
category_probs = Translation&Rotation:0.3 Permutation&Block:0.3 Strain:0.3 Conformer:0.1
```
