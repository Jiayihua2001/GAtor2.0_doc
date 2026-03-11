# Flexible Molecules

GAtor supports crystal structure prediction for molecules with conformational flexibility — molecules that can adopt different shapes by rotating around single bonds (torsion angles).

---

## Enabling Flexible Mode

```ini
[run_settings]
flexible = TRUE

[conformer]
pre_calc_conformer_json_name = conformer.json
rmsd_threshold = 1.5
```

### The Conformer Pool

The `conformer.json` file contains pre-computed molecular conformers in ASE Atoms JSON format. Each conformer should have a `conformer_energy` property:

```python
from ase.io.jsonio import encode
import json

conformers = {}
for i, atoms in enumerate(conformer_list):
    atoms.info["conformer_energy"] = energy_list[i]
    conformers[f"conf_{i:03d}"] = encode(atoms)

with open("conformer.json", "w") as f:
    json.dump(conformers, f)
```

During initial pool preparation, GAtor will:

1. Re-compute conformer energies with the configured optimization module for consistency
2. Remove duplicate conformers (using pymatgen's `MoleculeMatcher`)
3. Assign a conformer ID to each molecule in each crystal structure
4. Save the deduplicated conformer pool to `unique_mol_dict.json`

### RMSD Threshold

The `rmsd_threshold` controls how strictly conformers are matched:

```ini
[conformer]
rmsd_threshold = 1.5    # Angstroms
```

If a molecule in a crystal doesn't match any known conformer within this threshold, GAtor logs a warning and flags it as a potential new conformer.

---

## Crossover with Flexible Molecules

### Torsion Angle Crossover

Use `torsion_angle_crossover` as the crossover module for flexible systems:

```ini
[modules]
crossover_module = torsion_angle_crossover
```

This module blends torsion angles between parent structures in addition to lattice and molecular position crossover.

### Symmetric Crossover with Torsion Blending

Alternatively, `symmetric_crossover` can blend torsion angles when the `[conformer]` section is present:

```ini
[modules]
crossover_module = symmetric_crossover

[crossover]
blend_torsion_prob = 0.5
```

---

## Mutation with Flexible Molecules

Two mutation types are specific to flexible molecules:

### Conformer Mutation

Replaces the molecular conformer with a different one from the conformer pool:

```ini
[mutation]
specific_mutations = Conformer_mutation Rand_trans Rand_rot Rand_strain
```

### Torsion Angle Mutation

Applies a random rotation to one or more torsion angles:

```ini
[mutation]
specific_mutations = Torsion_angle Rand_trans Rand_rot Rand_strain
```

---

## Conformer Energy Correction

When computing fitness, GAtor can account for the intramolecular energy difference between conformers. The conformer energy stored in each structure is subtracted from the total crystal energy to approximate the lattice energy:

$$E_{\text{lattice}} = E_{\text{crystal}} - \sum_i n_i \cdot E_{\text{conformer},i}$$

where $n_i$ is the number of molecules of conformer $i$ in the unit cell.
