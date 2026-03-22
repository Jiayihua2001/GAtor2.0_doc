# Optimization Modules

Optimization modules handle energy evaluation and structural relaxation. GAtor supports both machine-learning interatomic potentials (MLIPs) and density functional theory (DFT) backends.

---

## Module Selection

```ini
[modules]
optimization_module = UMA     # or: MACE, AIMNET2, FHI_aims, VASP
spe_module = UMA              # Single-point energy (can differ from optimization)
```

---

## MACE

[MACE](https://github.com/ACEsuit/mace) is a higher-order equivariant message-passing neural network potential. GAtor uses the pre-trained `mace-off` model.

```ini
[modules]
optimization_module = MACE

[MACE]
store_energy_names = mace mace_bfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500
```

| Option | Type | Description |
|--------|------|-------------|
| `store_energy_names` | list | Property names for SPE and relaxed energy |
| `relative_energy_thresholds` | list[float] | Reject if energy > min + threshold (eV) |
| `absolute_energy_thresholds` | list[float] | Reject if energy > absolute value (eV) |
| `reject_if_worst_energy` | list[bool] | Reject if worst energy in pool |
| `fmax` | float | Force convergence criterion (eV/A) |
| `steps` | int | Maximum optimizer steps |
| `save_trajectory` | bool | Save optimization trajectory |

**Relaxation method**: BFGS with `FrechetCellFilter` (cell + positions) and `FixSymmetry` constraint.

---

## UMA (FAIRChem)

[UMA](https://github.com/FAIR-Chem/fairchem) (Universal Machine-learning Atomistic) from Meta's FAIRChem project.

```ini
[modules]
optimization_module = UMA

[UMA]
store_energy_names = uma uma_lbfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500
head = omc
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `head` | string | `omc` | UMA model head (e.g., `omc`, `omat`) |

**Relaxation method**: LBFGS with `FrechetCellFilter` and `FixSymmetry`.

---

## AIMNet2

[AIMNet2](https://github.com/isayevlab/AIMNet2) is a neural network potential for organic molecules.

```ini
[modules]
optimization_module = AIMNET2

[AIMNET2]
checkpoint_path = /path/to/aimnet2_checkpoint.pt
store_energy_names = aimnet2 aimnet2_bfgs
relative_energy_thresholds = 3 3
reject_if_worst_energy = TRUE TRUE
fmax = 0.01
steps = 1500
```

| Option | Type | Description |
|--------|------|-------------|
| `checkpoint_path` | string | Path to AIMNet2 model checkpoint |

---

## FHI-aims

[FHI-aims](https://fhi-aims.org/) is an all-electron density functional theory (DFT) code that uses numeric atom-centered orbital (NAO) basis sets. It provides high-accuracy results for molecular crystals with support for dispersion corrections (MBD) and periodic boundary conditions.

### Configuration

GAtor interfaces with FHI-aims through an `aims.json` settings file that specifies DFT parameters. Pre-configured settings are provided in `examples/00_setup/DFT_settings/FHI-aims/aims.json`.

```ini
[modules]
optimization_module = FHI-aims

[FHI-aims]
execute_command = srun
path_to_aims_executable = /path/to/aims.x
aims_settings_path = ./aims.json
k_density = 25
fmax = 0.01
steps = 100
store_energy_names = aims aims_bfgs
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
save_failed_calc = TRUE
```

| Option | Type | Description |
|--------|------|-------------|
| `execute_command` | string | `srun` or `mpirun` |
| `path_to_aims_executable` | string | Absolute path to FHI-aims binary |
| `aims_settings_path` | string | Path to `aims.json` with DFT settings |
| `k_density` | int | k-point density for Brillouin zone sampling |
| `fmax` | float | Force convergence criterion (eV/A) |
| `steps` | int | Maximum geometry optimization steps |
| `save_failed_calc` | bool | Save failed calculations for debugging |

### DFT Settings (`aims.json`)

The `aims.json` file specifies FHI-aims control parameters:

```json
{
    "species_dir": "/path/to/fhi-aims/species_defaults/defaults_2020/light",
    "xc": "pbe",
    "spin": "none",
    "relativistic": "atomic_zora scalar",
    "charge": 0,
    "occupation_type": "gaussian 0.01",
    "mixer": "pulay",
    "n_max_pulay": 8,
    "charge_mix_param": 0.2,
    "sc_accuracy_rho": 1e-5,
    "sc_accuracy_eev": 0.001,
    "sc_accuracy_etot": 1e-6,
    "sc_accuracy_forces": 1e-4,
    "sc_iter_limit": 10000,
    "KS_method": "parallel",
    "empty_states": 6,
    "basis_threshold": 1e-5,
    "compute_forces": true,
    "many_body_dispersion": " "
}
```

!!! tip "Basis Set"
    The `species_dir` should point to your FHI-aims installation's species defaults. The `light` tier is recommended for GA screening; switch to `tight` for final refinements.

!!! tip "Dispersion Corrections"
    The `many_body_dispersion` keyword enables MBD dispersion corrections, which are important for molecular crystal energetics.

---

## VASP

[VASP](https://www.vasp.at/) is a plane-wave DFT code. GAtor interfaces with VASP through a `vasp.json` settings file. Pre-configured settings are provided in `examples/00_setup/DFT_settings/VASP/`.

```ini
[modules]
optimization_module = VASP

[VASP]
execute_command = srun
ncore = 128
path_to_vasp_executable = vasp_ncl
store_energy_names = vasp_SPE vasp_bfgs
relative_energy_thresholds = 1000 3
reject_if_worst_energy = FALSE FALSE
fmax = 0.01
steps = 100
energy_settings_path = ./vasp.json
pp_setups = minimal
```

| Option | Type | Description |
|--------|------|-------------|
| `execute_command` | string | `srun` or `mpirun` |
| `ncore` | int | Number of CPUs per replica |
| `path_to_vasp_executable` | string | Path to VASP binary |
| `energy_settings_path` | string | Path to `vasp.json` with INCAR parameters |
| `pp_setups` | string | Pseudopotential setup (e.g., `minimal`) |

!!! note "Environment Setup"
    Set `VASP_PP_PATH` to your pseudopotential directory before running:
    ```bash
    export VASP_PP_PATH=/path/to/Pseudopotentials
    ```

---

## Energy Thresholds

All optimization modules support energy thresholds to reject high-energy structures early:

- **Relative threshold**: Reject if `energy > min_energy_in_pool + threshold`
- **Absolute threshold**: Reject if `energy > absolute_value`
- **Worst energy**: Reject if structure has the highest energy in the pool

These are applied per control file / optimization step, allowing different thresholds for SPE vs. full relaxation.
