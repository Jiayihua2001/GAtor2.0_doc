# Tutorial 1: Setup & Preparation

Choose a calculator, configure parallelization, prepare initial structures, and determine the critical **specific radius proportion** (SR) parameter.

---

## Step 1: Choose a Calculator

| Calculator | Type | Speed | Hardware | Install |
|---|---|---|---|---|
| **UMA** | MLIP | ~30 min | GPU | `pip install -e .[uma] --no-build-isolation` |
| **MACE** | MLIP | ~30 min | GPU | `pip install -e .[mace] --no-build-isolation` |
| **AIMNet2** | MLIP | ~30 min | GPU | `pip install -e .[aimnet2] --no-build-isolation` |
| **FHI-aims** | DFT (all-electron) | 1–5 hours | CPU | External binary |
| **VASP** | DFT | 1–5 hours | CPU | External binary |

!!! tip "Recommendation"
    Start with **UMA or MACE** for rapid prototyping, then switch to DFT for production-quality results if needed.

Set the calculator in `ui.conf`:

```ini
[modules]
optimization_module = UMA   # Options: UMA, MACE, AIMNET2, FHI-aims, VASP
```

Each calculator has its own configuration section — see `examples/00_setup/calculator.conf` for the full reference.

**Two-stage energy filtering:** GAtor evaluates structures in two stages. The first threshold (e.g., 1000 eV) is a generous filter after single-point evaluation; the second (e.g., 3 eV) is applied after full geometry optimization.

---

## Step 2: Configure Parallelization

=== "GPU (MLIP)"

    Each GPU provides one GA replica. A typical GPU node has 4 GPUs = 4 replicas.

    ```ini
    [parallel_settings]
    parallelization_method = srun
    run_on_gpu = TRUE
    replicas_per_node = 4
    ```

    | Nodes | Replicas | Use Case |
    |---|---|---|
    | 1 | 4 | Quick test (`-q debug`, 30 min) |
    | 2 | 8 | Standard production |
    | 4 | 16 | Large-scale exploration |

=== "CPU (DFT)"

    Each replica gets many CPUs for the ab initio calculation.

    ```ini
    [parallel_settings]
    parallelization_method = srun
    run_on_gpu = FALSE
    replicas_per_node = 1
    cpus_per_replica = 128
    ```

Ready-to-use SLURM scripts are in `examples/00_setup/` (`submit_gpu.sh` and `submit_cpu.sh`).

---

## Step 3: Prepare `structures.json`

### Using Genarris (Recommended)

Generate the initial pool with [Genarris 3.0](https://github.com/Yi5817/Genarris), then copy the output:

```bash
cp /path/to/genarris/output/structures.json ./structures.json
```

### From CIF or POSCAR Files

```bash
python examples/01_prepare/prepare_structures.py /path/to/cif_directory \
    --output structures.json

# POSCAR format:
python examples/01_prepare/prepare_structures.py /path/to/poscar_directory \
    --format poscar --output structures.json
```

### Cocrystals & Z' > 1

For multi-component systems, consistent atom ordering is critical:

```bash
python examples/01_prepare/prepare_structures.py /path/to/cifs \
    --output structures.json --reorder --ui-conf /path/to/ui.conf
```

Also set `reorder_initial_pool = TRUE` in `[initial_pool]` (see [Tutorial 3](cocrystal.md)).

---

## Step 4: Determine SR

The **specific radius proportion** (SR) defines the minimum allowed distance between atoms of different molecules as a fraction of their van der Waals radii. It is the most critical parameter.

### Run Pool Analysis

```bash
python examples/01_prepare/analyze_pool.py \
    --ui-conf ui.conf \
    --pool-path ./structures.json \
    --pool-format structures_json \
    --energy-key <energy-key> \
    --output-dir pool_analysis_results \
    --exp-file /path/to/experimental.cif   # optional
```

**Output:**

- `pool_analysis.csv` — Per-structure SR, energy, volume, space group
- `pool_analysis.png` — 4-panel figure: SR histogram, energy vs volume, space group distribution, energy histogram

### Auto-Recommend SR

```bash
python examples/01_prepare/recommend_sr.py pool_analysis_results/pool_analysis.csv

# Custom target pass rate (default: 0.80):
python examples/01_prepare/recommend_sr.py pool_analysis.csv --target-pass-rate 0.75
```

### Choosing SR Manually

Pick the SR where **~80% of the pool passes**:

| Pool Pass Rate | Action |
|---|---|
| > 95% | SR may be too low — consider increasing |
| 75–90% | Good range for most systems |
| < 60% | SR is too high — lower it or check your pool |

!!! important "Experimental Structure SR"
    If you have an experimental structure, ensure your chosen SR is **at or below** the experimental SR — otherwise the GA cannot find it.

### Minimal `ui.conf` for Pool Analysis

```ini
[modules]
optimization_module = UMA

[run_settings]
num_molecules = 4

[parallel_settings]
parallelization_method = serial
replicas_per_node = 1

[cell_check_settings]
specific_radius_proportion = 0.75
```

---

## Next Steps

Choose your crystal type and proceed to the corresponding tutorial:

| Crystal Type | Tutorial |
|---|---|
| Rigid molecule | [Tutorial 2: Rigid Molecule](rigid-mlip.md) |
| Multi-component cocrystal | [Tutorial 3: Cocrystal](cocrystal.md) |
| Flexible molecule | [Tutorial 4: Flexible Molecule](flexible.md) |
| PXRD-guided | [Tutorial 5: PXRD-Assisted](pxrd-assisted.md) |
