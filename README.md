# learning_openmc

Experiments and learnings with [OpenMC](https://openmc.org/) — an open-source Monte Carlo particle transport simulation code.

## Environment Setup

This project uses [uv](https://docs.astral.sh/uv/) to manage the Python environment. OpenMC itself is built from source because there are no pre-built pip wheels, and the conda-forge package only supports x86_64 (Rosetta) on macOS — not native ARM.

### Prerequisites

macOS with Apple Silicon and [Homebrew](https://brew.sh/) installed.

### 1. Install system dependencies

```bash
brew install llvm cmake hdf5 libomp libpng
```

- **llvm** — C/C++ compiler with OpenMP support (Xcode's clang does not support OpenMP)
- **cmake** — build system for the C++ engine
- **hdf5** — data format used by OpenMC for cross-section data and output
- **libomp** — OpenMP runtime for multithreaded simulations
- **libpng** — plotting support

### 2. Clone and build the OpenMC C++ engine

```bash
git clone --recurse-submodules https://github.com/openmc-dev/openmc.git ~/openmc-source
cd ~/openmc-source
mkdir build && cd build

export CXX=/opt/homebrew/opt/llvm/bin/clang++
export CC=/opt/homebrew/opt/llvm/bin/clang
export SDKROOT=$(xcrun --show-sdk-path)

cmake -DCMAKE_INSTALL_PREFIX=$HOME/.local \
      -DHDF5_ROOT=/opt/homebrew/opt/hdf5 \
      -DCMAKE_PREFIX_PATH=/opt/homebrew/opt/llvm \
      -DCMAKE_OSX_SYSROOT=$SDKROOT \
      -DCMAKE_CXX_FLAGS="--sysroot=$SDKROOT -I/opt/homebrew/opt/libomp/include" \
      -DCMAKE_C_FLAGS="--sysroot=$SDKROOT" \
      ..

make -j$(sysctl -n hw.ncpu)
make install
```

This installs the `openmc` executable to `~/.local/bin/`. The `--sysroot` flags are needed because brew's LLVM cannot find the macOS SDK headers on its own.

### 3. Set up the Python environment

```bash
cd /path/to/this/repo

# Create the virtual environment and install project dependencies
uv sync

# Install the OpenMC Python API from the local source clone
HDF5_DIR=/opt/homebrew/opt/hdf5 uv pip install ~/openmc-source
```

The OpenMC Python package brings in its own dependencies: numpy, scipy, h5py, matplotlib, pandas, lxml, uncertainties, ipython, and endf.

### 4. Download nuclear cross-section data

OpenMC requires nuclear data libraries to run simulations. The ENDF/B-VIII.0 library (~3.2 GB compressed) is recommended:

```bash
mkdir -p ~/openmc-data
cd ~/openmc-data
curl -L -o endfb-viii.0-hdf5.tar.xz https://anl.box.com/shared/static/uhbxlrx7hvxqw27psymfbhi7bx7s6u6a.xz
tar xf endfb-viii.0-hdf5.tar.xz
```

This extracts neutron, photon, and thermal scattering data in HDF5 format along with a `cross_sections.xml` index file.

Other libraries (ENDF/B-VIII.1, JEFF 4.0, JENDL-5) are available at [openmc.org/data](https://openmc.org/data/).

### 5. Configure environment variables

Add to `~/.zshrc`:

```bash
# OpenMC
export PATH="$HOME/.local/bin:$PATH"
export OPENMC_CROSS_SECTIONS="$HOME/openmc-data/endfb-viii.0-hdf5/cross_sections.xml"
```

Then reload: `source ~/.zshrc`

### Verify the installation

```bash
# Check the Python API
uv run python -c "import openmc; print(openmc.__version__)"

# Check that cross-section data is found
OPENMC_CROSS_SECTIONS=$HOME/openmc-data/endfb-viii.0-hdf5/cross_sections.xml \
uv run python -c "
import openmc
mat = openmc.Material()
mat.add_nuclide('U235', 1.0)
print('OpenMC is working')
"
```

## Usage

```bash
# Run a Python script
uv run python my_script.py

# Start a Jupyter notebook
uv run jupyter notebook
```

## Directory Layout

| Path | Contents |
|------|----------|
| `~/openmc-source/` | OpenMC source code (git clone) |
| `~/.local/bin/openmc` | Compiled C++ executable (native ARM64) |
| `~/openmc-data/endfb-viii.0-hdf5/` | Nuclear cross-section data |
| This repo `.venv/` | Python virtual environment (managed by uv) |
