# Fitness Modules

Fitness modules evaluate the quality of crystal structures, assigning a numerical score that drives selection pressure.

---

## Module Selection

```ini
[modules]
fitness_module = standard_energy
```

---

## Standard Energy Fitness

Ranks structures by total energy. Lower energy = higher fitness.

```ini
[modules]
fitness_module = standard_energy

[fitness]
energy_name = energy
```

The fitness is computed as a normalized relative value:

$$\rho = \frac{E_{\max} - E}{E_{\max} - E_{\min}}$$

where $E_{\max}$ and $E_{\min}$ are the maximum and minimum energies in the pool.

| Option | Type | Description |
|--------|------|-------------|
| `energy_name` | string | Property name to use as energy |

---

## VC Energy Fitness (Multi-Objective)

Combines energy with a secondary criterion (e.g., PXRD similarity) for multi-objective optimization.

```ini
[modules]
fitness_module = vc_energy_fitness

[fitness]
energy_name = energy
pxrd_file = /path/to/experimental.xy
pxrd_cell = 10.37 12.61 13.85 90 90.27 90
pxrd_scaling_factor = 0.25
```

| Option | Type | Description |
|--------|------|-------------|
| `pxrd_file` | string | Path to experimental PXRD pattern (.xy format) |
| `pxrd_cell` | string | Cell parameters: `a b c alpha beta gamma` |
| `pxrd_scaling_factor` | float | Weight of PXRD similarity (0-1) |

The combined fitness blends energy ranking with PXRD similarity:

$$F = (1 - \lambda) \cdot F_{\text{energy}} + \lambda \cdot F_{\text{PXRD}}$$

where $\lambda$ is the `pxrd_scaling_factor`.

!!! note "critic2 Required"
    PXRD pattern calculation requires [critic2](https://aoterodelaroza.github.io/critic2/) to be installed and in your PATH.

---

## Shared Fitness (Cluster-Based)

Used with clustering to implement fitness sharing — structures in crowded regions of the landscape receive reduced fitness, promoting diversity:

$$F_{\text{shared}} = \frac{F}{\text{cluster\_members}}$$

This is applied automatically when clustering is enabled.

---

## Optimization Style

Control whether the GA minimizes or maximizes the fitness criterion:

```ini
[run_settings]
optimization_style = minimize    # or: maximize
```
