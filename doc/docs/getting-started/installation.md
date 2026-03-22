# Installation

This guide walks through setting up GAtor on an HPC system with GPU support.

## Prerequisites

- **Conda** (Miniconda or Anaconda)
- **SWIG** (for compiling C extensions)
- **MPI compiler** (`mpicc` / `mpiicc`) if using MPI parallelization
- **CUDA** (optional, for GPU-accelerated MLIPs)

---

## Step 1: Load an MPI Module

GAtor works with **any** MPI implementation (Intel MPI, OpenMPI, MPICH, etc.).
The only requirement is that `mpi4py` is compiled from source against the
**same** MPI library that will be used at runtime.

```bash
# Load whichever MPI is available on your system, e.g.:
module load intel intelmpi   # Intel MPI
# module load openmpi        # or OpenMPI
# module load mpich           # or MPICH
```

## Step 2: Create a Conda Environment

```bash
conda create -n gator python=3.10 swig numpy
conda activate gator
```

!!! tip "Faster alternative: mamba"
    You can use [`mamba`](https://mamba.readthedocs.io) as a faster drop-in replacement for `conda`:
    ```bash
    mamba create -n gator python=3.10 swig numpy
    mamba activate gator
    ```

## Step 3: Install mpi4py

Build `mpi4py` from source to match your system's MPI library:

```bash
MPICC=$(which mpicc) pip install --no-cache-dir --no-binary mpi4py mpi4py
```

!!! warning "Do not skip `--no-binary`"
    Pre-built `mpi4py` wheels ship with generic MPI bindings that cause struct-size mismatches and segfaults at runtime. Building from source ensures the C struct layouts match your system's MPI library exactly.

!!! warning "MPI Consistency"
    Ensure that `mpi4py`, your MPI library, and your DFT code (if applicable) all use the **same MPI implementation** (e.g., all use Cray MPICH or all use OpenMPI). See the [mpi4py documentation](https://mpi4py.readthedocs.io/en/stable/install.html) for details.

## Step 4: Clone GAtor

```bash
git clone https://github.com/Jiayihua2001/GAtor.git
cd GAtor
```

## Step 5: Get the CGenarris Submodule

CGenarris is the C extension for random crystal structure generation:

```bash
cd gator/
git clone https://github.com/Jiayihua2001/cgenarris.git
cd ..
```

## Step 6: Install GAtor

```bash
# Install GAtor in editable mode
pip install -e . --no-build-isolation

# Install the rpack subpackage
cd gator/cgenarris/src/rpack
pip install -e . --no-build-isolation
cd ../../../..
```

!!! note "`--no-build-isolation` is required"
    This flag ensures the build uses the `mpi4py` and `numpy` already installed in your environment, rather than downloading fresh copies that may not match your MPI.

## Step 7: Install Energy Calculators

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

    No additional Python packages required — just ensure the executable is accessible on your HPC system.

    **FHI-aims** is an all-electron density functional theory (DFT) code that uses numeric atom-centered orbital basis sets. GAtor interfaces with FHI-aims via an `aims.json` settings file specifying DFT parameters (exchange-correlation functional, dispersion corrections, convergence criteria, etc.). Pre-configured FHI-aims settings are provided in `examples/00_setup/DFT_settings/FHI-aims/aims.json`.

    **VASP** is a plane-wave DFT code. Configure it via a `vasp.json` file and ensure `VASP_PP_PATH` points to your pseudopotential directory. See `examples/00_setup/DFT_settings/VASP/` for reference settings.

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
