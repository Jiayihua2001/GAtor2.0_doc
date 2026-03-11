# File Formats

## GAtor Structure JSON

Individual structure files in the pool (e.g., `tmp/pool_0/<id>.json`) use GAtor's internal JSON format:

```json
{
  "struct_id": "a1b2c3d4e5f6g7h",
  "properties": {
    "lattice_vector_a": [10.0, 0.0, 0.0],
    "lattice_vector_b": [0.0, 12.0, 0.0],
    "lattice_vector_c": [0.0, 0.0, 8.0],
    "energy": -125.432,
    "space_group": 14,
    "a": 10.0, "b": 12.0, "c": 8.0,
    "alpha": 90.0, "beta": 90.0, "gamma": 90.0,
    "unit_cell_volume": 960.0
  },
  "geometry": [
    {"x": 1.0, "y": 2.0, "z": 3.0, "element": "C", "spin": 0, "charge": 0, "fixed": false},
    ...
  ]
}
```

## ASE Atoms JSON (`structures.json`)

Input structures use ASE's JSON serialization:

```python
from ase.io.jsonio import encode, decode
import json

# Write
atoms_dict = {"struct_001": encode(atoms1), "struct_002": encode(atoms2)}
with open("structures.json", "w") as f:
    json.dump(atoms_dict, f)

# Read
with open("structures.json") as f:
    atoms_dict = json.load(f)
atoms = decode(atoms_dict["struct_001"])
```

## Energy Hierarchy File

The `energy_hierarchy_*.dat` file is a fixed-width table ranking all structures:

```
Rank   ID     Energy       Volume       SG     Mutation    ParentA     ParentB     ...
1      a1b2   -125.432     960.0        14     Rand_rot    c3d4        e5f6        ...
2      g7h8   -125.210     955.3        14     Rand_trans  i9j0        k1l2        ...
```

Columns include: rank, structure ID, energy, volume, lattice parameters, space group, mutation type, parent IDs, cluster assignment, and conformer IDs.

## GA Duplicates File (`GA_duplicates.dat`)

Tracks qualifying duplicate structures that were saved during the GA run:

```
new_struct_id    matched_struct_id    rmsd    comparison_stage
a1b2c3d4         e5f6g7h8             0.35    post_relaxation
i9j0k1l2         m3n4o5p6             0.28    pre_relaxation
```

Columns:

- **new_struct_id**: ID of the newly generated structure
- **matched_struct_id**: ID of the existing pool structure it matched
- **rmsd**: RMSD distance between the two structures (only saved when >= 0.2 A)
- **comparison_stage**: Whether the duplicate was detected at `pre_relaxation` or `post_relaxation`

Qualifying duplicates are also saved as structure files in the `duplicates/` pool directory.

---

## Conformer JSON

Conformer pool files map conformer IDs to ASE Atoms:

```json
{
  "conf_000": "<ASE encoded atoms with conformer_energy in info>",
  "conf_001": "...",
  ...
}
```

## Molecule Input File (`mol.in`)

Reference molecular geometry in ASE-readable format (typically extended XYZ):

```
12
Properties=species:S:1:pos:R:3
C  0.000  0.000  0.000
C  1.400  0.000  0.000
...
```
