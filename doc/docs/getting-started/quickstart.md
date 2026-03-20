# Quick Start

Run your first crystal structure prediction in **5 minutes** using the included Uracil example.

---

## Prerequisites

- GAtor is [installed](installation.md) and the `gator` environment is activated
- Access to a GPU node (1 node is enough for this test)

## 1. Copy the Example

```bash
cp -r /path/to/GAtor/examples/02_quick_start/Uracil/MLIP my_uracil_run
cd my_uracil_run
```

This gives you everything you need: `ui.conf`, `submit.sh`, `structures.json`, and an experimental CIF for validation.

## 2. Edit Two Files

**`ui.conf`** — Set the path to the experimental structure:

```ini
[initial_pool]
exp_path = /absolute/path/to/my_uracil_run/1278441.cif
```

**`submit.sh`** — Set your allocation and uncomment the environment activation:

```bash
#SBATCH -A your_allocation            # <- your SLURM account
# conda activate gator                # <- uncomment this line
```

## 3. Submit

```bash
sbatch submit.sh
tail -f GAtor.log      # watch progress
```

The run takes **~30 minutes** on 1 GPU node (4 replicas) using the `debug` queue.

## 4. Check Results

```bash
# Top 20 structures by energy
head -20 tmp/energy_hierarchy_*.dat

# Did the GA find the experimental structure?
cat tmp/exp_match_record.dat
```

The lowest-energy structures in `energy_hierarchy_*.dat` are your best predictions. `Added = 0` means initial pool; `Added > 0` means GA-generated.

---

## What Just Happened?

GAtor ran through 200 GA iterations using the UMA machine-learning potential:

1. Loaded ~100 initial structures from `structures.json`
2. Evolved them through crossover, mutation, and selection
3. Filtered duplicates and unphysical structures
4. Ranked all structures by energy

The configuration used default parameters suitable for rigid molecules. See [Tutorial 2](../tutorials/rigid-mlip.md) for a detailed walkthrough of every setting.

---

## Next Steps

| What | Where |
|---|---|
| Understand the config file | [Tutorial 2: Rigid Molecule](../tutorials/rigid-mlip.md) |
| Prepare your own structures | [Tutorial 1: Setup & Preparation](../tutorials/setup.md) |
| Try a different crystal type | [Tutorials Overview](../tutorials/index.md) |
| Analyze results properly | [Tutorial 6: Post-Analysis](../tutorials/post-analysis.md) |
