# Initial Pool

The initial pool is the starting set of crystal structures from which the GA evolves new candidates. A diverse, well-constructed initial pool is critical for efficient exploration.

---

## Automatic Pool Preparation

The easiest approach is to set `prepare_initial_pool = TRUE` and provide a `structures.json` file:

```ini
[initial_pool]
prepare_initial_pool = TRUE
user_structures_dir = initial_pool
stored_energy_name = uma_lbfgs
```

GAtor will:

1. Read structures from `structures.json` (ASE Atoms JSON format)
2. Optimize each structure using the configured optimization module
3. Optionally remove structures matching an experimental reference
4. Detect conformers (if flexible molecule mode is enabled)
5. Perform ASU partner swap for cocrystals
6. Analyze space groups
7. Write optimized structures to `initial_pool/`

### Preparing `structures.json`

The `structures.json` file is a JSON dictionary mapping structure IDs to ASE Atoms objects:

```python
from ase.io import read
from ase.io.jsonio import encode
import json

structures = {}
for i, cif_file in enumerate(cif_files):
    atoms = read(cif_file)
    structures[f"struct_{i:04d}"] = encode(atoms)

with open("structures.json", "w") as f:
    json.dump(structures, f)
```

!!! tip "Structure Sources"
    - **PyGenarris** — Random structure generation (included with GAtor)
    - **CSD** — Cambridge Structural Database
    - **AIRSS** — Ab Initio Random Structure Searching
    - **Manual** — Convert CIF files to ASE format

### Removing Experimental Matches

To exclude structures that match a known experimental structure:

```ini
[initial_pool]
remove_match = TRUE
exp_path = /path/to/experimental.cif
```

### Reordering Atoms

If atomic ordering in your input structures doesn't match the reference molecule:

```ini
[initial_pool]
reorder_initial_pool = TRUE
```

### Restart from Checkpoint

If pool preparation was interrupted, resume from the checkpoint:

```ini
[initial_pool]
restart = TRUE
```

This reads `save.json` and continues from where it left off.

---

## Manual Pool Preparation

If you prefer to prepare the pool yourself:

1. Set `prepare_initial_pool = FALSE`
2. Place optimized structure JSON files in the `initial_pool/` directory
3. Each file should be a GAtor Structure JSON containing geometry and energy properties

---

## Pool Analysis

After filling the pool, you can run an automatic analysis:

```ini
[GAtor_master]
analyze_initial_pool = TRUE

[pool_analysis]
sr_min = 0.60
sr_max = 1.05
sr_step = 0.01
exp_file = /path/to/experimental.cif
```

This generates plots and statistics in `tmp/pool_analysis/`:
- Energy distribution
- Space group distribution
- Specific radius (SR) analysis
- Comparison with experimental structure
