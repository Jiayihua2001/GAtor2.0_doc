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

[FHI-aims](https://fhi-aims.org/) is an all-electron DFT code.

```ini
[modules]
optimization_module = FHI_aims

[FHI-aims]
execute_command = srun
path_to_aims_executable = /path/to/aims.x
control_in_directory = control
control_in_filelist = control.in.SPE control.in.FULL
store_energy_names = energy_SPE energy_FULL
relative_energy_thresholds = 1000 1000
reject_if_worst_energy = FALSE FALSE
save_failed_calc = TRUE
save_successful_calc = TRUE
save_aims_output = TRUE
monitor_execution = FALSE
update_poll_times = 1200 1200
```

| Option | Type | Description |
|--------|------|-------------|
| `execute_command` | string | `srun` or `mpirun` |
| `path_to_aims_executable` | string | Absolute path to aims binary |
| `control_in_directory` | string | Directory with control.in files |
| `control_in_filelist` | list | Control files (applied sequentially) |
| `save_aims_output` | bool | Keep aims output files |
| `monitor_execution` | bool | Monitor job progress |
| `update_poll_times` | list[int] | Timeout per control file (seconds) |

!!! tip "Multi-step Relaxation"
    Use multiple control files for a tiered relaxation strategy: first a cheap SPE, then a full relaxation. Structures failing the SPE threshold are discarded without running the expensive relaxation.

---

## VASP

[VASP](https://www.vasp.at/) is a plane-wave DFT code.

```ini
[modules]
optimization_module = VASP

[VASP]
path_to_vasp_executable = /path/to/vasp_std
store_energy_names = vasp_spe vasp_relax
relative_energy_thresholds = 1000 1000
reject_if_worst_energy = FALSE FALSE
```

---

## Energy Thresholds

All optimization modules support energy thresholds to reject high-energy structures early:

- **Relative threshold**: Reject if `energy > min_energy_in_pool + threshold`
- **Absolute threshold**: Reject if `energy > absolute_value`
- **Worst energy**: Reject if structure has the highest energy in the pool

These are applied per control file / optimization step, allowing different thresholds for SPE vs. full relaxation.
