# Tutorial 3: Flexible Molecule

This tutorial covers crystal structure prediction for molecules with conformational flexibility.

---

## Scenario

- **System**: A flexible organic molecule with rotatable bonds
- **Z = 4**: 4 molecules per unit cell
- **Backend**: MACE on GPU
- **Conformers**: Pre-computed set of molecular conformers

## Step 1: Prepare Conformer Pool

Generate molecular conformers using RDKit, CREST, or other tools, then save as `conformer.json`:

```python
from ase.io import read
from ase.io.jsonio import encode
import json

conformers = {}
for i, xyz_file in enumerate(conformer_files):
    atoms = read(xyz_file)
    atoms.info["conformer_energy"] = energies[i]  # in eV
    conformers[f"conf_{i:03d}"] = encode(atoms)

with open("conformer.json", "w") as f:
    json.dump(conformers, f)
```

!!! important "Conformer Energies"
    GAtor will re-compute conformer energies using the configured optimization module during pool preparation, ensuring consistency between crystal and molecular energies.

## Step 2: Configuration

```ini
[run_settings]
num_molecules = 4
flexible = TRUE

[modules]
optimization_module = MACE
crossover_module = symmetric_crossover
mutation_module = standard_mutation

[conformer]
pre_calc_conformer_json_name = conformer.json
rmsd_threshold = 1.5

[crossover]
crossover_probability = 0.75
blend_torsion_prob = 0.5

[mutation]
stand_dev_trans = 3.0
stand_dev_rot = 30
stand_dev_strain = 0.3
specific_mutations = Rand_trans Rand_rot Rand_strain Conformer_mutation Torsion_angle
```

## Step 3: Run

```bash
python /path/to/GAtor/gator/GAtor_master.py ui.conf
```

During initial pool preparation, GAtor will:

1. Optimize each crystal structure
2. Re-compute conformer energies with MACE for consistency
3. Deduplicate conformers
4. Assign conformer IDs to each molecule in each crystal
5. Save the conformer pool to `unique_mol_dict.json`

## Step 4: Analyze

Each structure in the pool tracks its conformer assignments:

- `conformer_0_ID`: ID of the assigned conformer
- `conformer_0_energy`: Intramolecular energy of that conformer

---

## Cocrystal + Flexible

For flexible cocrystals, provide separate conformer files:

```ini
[conformer]
pre_calc_conformer_json_name = conformer_0.json conformer_1.json
```

Where `conformer_0.json` contains conformers for molecule type A and `conformer_1.json` for type B.
