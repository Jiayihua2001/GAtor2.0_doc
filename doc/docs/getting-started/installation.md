# Installation

This guide walks through setting up GAtor on an HPC system with GPU support.

## Prerequisites

- **Conda** (Miniconda or Anaconda)
- **SWIG** (for compiling C extensions)
- **MPI compiler** (`mpicc`) if using MPI parallelization
- **CUDA** (optional, for GPU-accelerated MLIPs)

---

## Step 1: Create a Conda Environment

```bash
conda create -n gator python=3.9 swig -y
conda activate gator
```

!!! tip "Python Version"
    GAtor requires Python 3.9 or later. Python 3.10–3.11 is recommended.

## Step 2: Install MPI Support (Optional)

If you plan to use MPI parallelization, install `mpi4py` after loading your system's MPI module:

```bash
# Load the MPI module (system-specific)
module load cray-mpich   # Example for Cray systems
# or: module load openmpi

# Install mpi4py with your MPI compiler
MPICC="$(which mpicc)" pip install mpi4py
```

!!! warning "MPI Consistency"
    Ensure that `mpi4py`, your MPI library, and your DFT code (if applicable) all use the **same MPI implementation** (e.g., all use Cray MPICH or all use OpenMPI).

## Step 3: Clone GAtor

```bash
git clone https://github.com/Jiayihua2001/GAtor.git
cd GAtor
```

## Step 4: Get the PyGenarris Submodule

PyGenarris is the C extension for random crystal structure generation:

```bash
cd gator/
git clone https://github.com/Yi5817/cgenarris.git
cd ..
```

## Step 5: Install GAtor

```bash
# Install GAtor in editable mode
pip install -e .

# Install the rpack subpackage
cd gator/cgenarris/src/rpack
pip install -e .
cd ../../../..
```

## Step 6: Install IBSLib

IBSLib provides structure I/O and dimer analysis utilities:

```bash
cd ibslib/
pip install -e .
cd ..
```

## Step 7: Install Additional Dependencies

Depending on your optimization backend, install the appropriate packages:

=== "MACE (Recommended)"

    ```bash
    pip install mace-torch
    ```

=== "UMA (FAIRChem)"

    ```bash
    pip install fairchem-core
    ```

=== "AIMNet2"

    ```bash
    pip install aimnet2calc
    ```

=== "FHI-aims / VASP"

    No additional Python packages required — just ensure the executable is accessible on your HPC system.

## Step 8: Set Environment Variables

Add the following to your shell configuration or job submission script:

```bash
export PYTHONPATH="${PYTHONPATH}:/path/to/GAtor/ibslib"
```

### Optional: critic2 (for PXRD fitness)

If using powder X-ray diffraction (PXRD) similarity as a fitness function, install [critic2](https://aoterodelaroza.github.io/critic2/installation/):

```bash
# After building critic2:
export PATH="/path/to/critic2/build/src:$PATH"
```

---

## Verify Installation

```bash
python -c "import gator; print('GAtor imported successfully')"
python -c "from gator.structures.structure import Structure; print('Structure module OK')"
python -c "from mace.calculators import mace_off; print('MACE available')"
```

!!! success "You're ready!"
    Proceed to the [Quick Start](quickstart.md) to run your first prediction.

---

## Troubleshooting

??? question "ImportError: No module named 'gator'"
    Make sure you installed GAtor in editable mode (`pip install -e .`) from the repository root, and that your Conda environment is activated.

??? question "MPI compiler not found during setup"
    The `setup.py` requires `mpicc` to be in your PATH. Load your MPI module first: `module load cray-mpich` (or equivalent).

??? question "CUDA out of memory with MACE/UMA"
    Reduce `replicas_per_node` in `[parallel_settings]`, or set `run_on_gpu = FALSE` to use CPU.
