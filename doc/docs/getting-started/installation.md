# Installation

This guide walks through setting up GAtor on an HPC system with GPU support.

## Prerequisites

- **Conda** (Miniconda or Anaconda)
- **SWIG** (for compiling C extensions)
- **MPI compiler** (`mpicc` / `mpiicc`) if using MPI parallelization
- **CUDA** (optional, for GPU-accelerated MLIPs)

---

## Step 1: Create a Conda Environment

```bash
conda create -n gator python=3.9 swig numpy -y
conda activate gator
```

!!! tip "Faster alternative: mamba"
    You can use [`mamba`](https://mamba.readthedocs.io) as a faster drop-in replacement for `conda`:
    ```bash
    mamba create -n gator python=3.9 swig numpy -y
    mamba activate gator
    ```

## Step 2: Install MPI Support

Load your system's MPI module, then install `mpi4py`:

```bash
# Load the MPI module (system-specific)
module load cray-mpich   # Cray systems (e.g., Perlmutter)
# or: module load openmpi

# Install mpi4py with your MPI compiler
MPICC=$(which mpiicc) pip install mpi4py==3.1.6
```

!!! warning "MPI Consistency"
    Ensure that `mpi4py`, your MPI library, and your DFT code (if applicable) all use the **same MPI implementation** (e.g., all use Cray MPICH or all use OpenMPI). See the [mpi4py documentation](https://mpi4py.readthedocs.io/en/stable/install.html) for details.

## Step 3: Clone GAtor

```bash
git clone https://github.com/Jiayihua2001/GAtor.git
cd GAtor
```

## Step 4: Get the CGenarris Submodule

CGenarris is the C extension for random crystal structure generation:

```bash
cd gator/
git clone https://github.com/Yi5817/cgenarris_dev.git cgenarris
cd ..
```

## Step 5: Install GAtor

```bash
# Install GAtor in editable mode
pip install -e . --no-build-isolation

# Install the rpack subpackage
cd gator/cgenarris/src/rpack
pip install -e . --no-build-isolation
cd ../../../..
```

## Step 6: Install Energy Calculators

GAtor supports various energy calculators through the [ASE Calculator](https://wiki.fysik.dtu.dk/ase/) interface. Install the one(s) you need:

=== "UMA (FAIRChem) — Recommended"

    ```bash
    pip install -e .[uma] --no-build-isolation
    ```

    !!! warning "HuggingFace Access"
        UMA models are gated. You need a HuggingFace account with access to the [UMA model repository](https://huggingface.co/facebook/UMA). Set `HF_HUB_OFFLINE=1` on compute nodes to avoid download attempts.

=== "MACE"

    ```bash
    pip install -e .[mace] --no-build-isolation
    ```

=== "AIMNet2"

    ```bash
    pip install -e .[aimnet2] --no-build-isolation
    ```

=== "FHI-aims / VASP (DFT)"

    No additional Python packages required — just ensure the executable is accessible on your HPC system. See `examples/00_setup/calculator.conf` for detailed calculator configuration.

!!! tip "Calculator Configuration Reference"
    See [`examples/00_setup/calculator.conf`](https://github.com/Jiayihua2001/GAtor/blob/main/examples/00_setup/calculator.conf) for detailed settings for each calculator backend.

### Optional: critic2 (for PXRD Fitness)

If using powder X-ray diffraction (PXRD) similarity as a fitness function (VC-PWDF metric), build and install [critic2](https://aoterodelaroza.github.io/critic2/):

```bash
git clone https://github.com/aoterodelaroza/critic2.git
cd critic2
mkdir build && cd build
cmake ..
make -j$(nproc)
cd ../..
```

Add critic2 to your PATH:

```bash
export CRITIC_HOME=/path/to/critic2
export PATH="$CRITIC_HOME/build/src:$PATH"
```

!!! tip "NERSC Users"
    On Perlmutter, you may need to load CMake first: `module load cmake`.

??? question "critic2 build fails with missing Fortran compiler"
    critic2 requires a Fortran compiler. Load one with `module load gcc` or `module load PrgEnv-gnu` before building.

### Optional: RDKit (for Conformer Generation)

If using flexible molecule mode, RDKit is useful for generating conformer pools:

```bash
conda install -c conda-forge rdkit
```

---

## Verify Installation

```bash
# Core package
python -c "import gator; print('GAtor imported successfully')"
python -c "from gator.structures.structure import Structure; print('Structure module OK')"

# Calculator backends (check the ones you installed)
python -c "from fairchem.core import OCPCalculator; print('UMA available')"
python -c "from mace.calculators import mace_off; print('MACE available')"

# CGenarris
python -c "import pygenarris; print('CGenarris available')"
```

!!! success "You're ready!"
    Proceed to the [Quick Start](quickstart.md) to run your first prediction.

---

## Troubleshooting

??? question "ImportError: No module named 'gator'"
    Make sure you installed GAtor in editable mode (`pip install -e . --no-build-isolation`) from the repository root, and that your Conda environment is activated.

??? question "MPI compiler not found during setup"
    The build requires `mpicc` to be in your PATH. Load your MPI module first: `module load cray-mpich` (or equivalent).

??? question "CUDA out of memory with MACE/UMA"
    Reduce `replicas_per_node` in `[parallel_settings]`, or set `run_on_gpu = FALSE` to use CPU.

??? question "CGenarris import fails"
    Ensure you cloned the correct repository (`cgenarris_dev.git`) into the `gator/` directory, and installed both the main package and the `rpack` subpackage with `--no-build-isolation`.
