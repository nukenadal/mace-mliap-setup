# MACE-MLIAP-LAMMPS Interface: Installation Guide (THEMMES Machines)

> **Status:** The ML-IAP interface is in **beta**. Report any issues to the MACE developers.

This guide walks through installing the MACE ML-IAP interface for LAMMPS on the THEMMES cluster (A100 GPUs, CUDA 12.9).

---

## 1. Load Modules

```bash
module load mpi/openmpi-x86_64
module load cuda/12.9
```

---

## 2. Set Up Python Environment

Using **venv**:

```bash
python -m venv ~/mace-mliap-env
source ~/mace-mliap-env/bin/activate
```

Or using **conda**:

```bash
conda create -n mace-mliap python=3.10 -y
conda activate mace-mliap
```

---

## 3. Install MACE and Dependencies

Download the latest MACE main branch and install:

```bash
# Download MACE (or git clone)
# Assumes you have mace-main/ in your workspace

pip install ./mace-main   # tested with 2026-01-09 snapshot

# PyTorch for CUDA 12.9
pip install torch==2.8.0 torchvision==0.23.0 torchaudio==2.8.0 \
  --index-url https://download.pytorch.org/whl/cu129
```

Then install cuEquivariance and other dependencies:

```bash
pip install cuequivariance-torch
pip install cuequivariance
pip install cuequivariance-ops-torch-cu12
pip install cupy-cuda12x
pip install cython
pip install numpy==2.0.0
```

---

## 4. Compile LAMMPS

### 4.1 Clone LAMMPS (develop branch)

```bash
git clone https://github.com/lammps/lammps.git
cd lammps
mkdir build-mliap
cd build-mliap
```

### 4.2 Create `kokkos-cuda.cmake`

Create the file `kokkos-cuda.cmake` in the build directory with the following content:

```cmake
# Preset that enables KOKKOS and selects CUDA compilation using nvcc_wrapper
set(PKG_KOKKOS ON CACHE BOOL "" FORCE)
set(Kokkos_ENABLE_SERIAL ON CACHE BOOL "" FORCE)
set(Kokkos_ENABLE_CUDA   ON CACHE BOOL "" FORCE)
set(Kokkos_ENABLE_CUDA_LAMBDA ON CACHE BOOL "" FORCE)

# A100 GPU architecture
set(Kokkos_ARCH_AMPERE80 ON CACHE BOOL "" FORCE)

get_filename_component(NVCC_WRAPPER_CMD ${CMAKE_CURRENT_SOURCE_DIR}/../lib/kokkos/bin/nvcc_wrapper ABSOLUTE)
set(CMAKE_CXX_COMPILER ${NVCC_WRAPPER_CMD} CACHE FILEPATH "" FORCE)

# If KSPACE is also enabled, use CUFFT for FFTs
set(FFT_KOKKOS "CUFFT" CACHE STRING "" FORCE)
set(Kokkos_ENABLE_DEPRECATION_WARNINGS OFF CACHE BOOL "" FORCE)
```

### 4.3 Configure with CMake

Make sure your Python environment is **activated** before running cmake:

```bash
/home/users/xia/workspace/codebase/cmake-3.24.4/bin/cmake -C kokkos-cuda.cmake \
  -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_INSTALL_PREFIX=$(pwd) \
  -D BUILD_MPI=ON \
  -D PKG_ML-IAP=ON \
  -D PKG_PYTHON=ON \
  -D MLIAP_ENABLE_PYTHON=ON \
  -D BUILD_SHARED_LIBS=ON \
  -D CMAKE_CXX_STANDARD=17 \
  ../cmake
```

> **Note:** We use a custom cmake path because the system cmake may be too old.

### 4.4 Build

```bash
make -j 8
```

If you hit CUDA compiler errors, try:

```bash
sed -i 's/ -Xcudafe --diag_suppress=unrecognized_pragma,--diag_suppress=128//' build/CMakeFiles/lmp.dir/flags.make7
make -j 8
```

---

## 5. Convert Your MACE Model

> **Important:** Run this on a **GPU node**, ideally on the same A100 architecture used for production.

```bash
python mace/cli/create_lammps_model.py your_trained_model.model --format=mliap
```

This produces `your_trained_model.model-mliap_lammps.pt`.

---

## 6. Example LAMMPS Input

```lammps
units           metal
atom_style      atomic
newton          on

read_data       structure.data

pair_style      mliap unified model-mliap_lammps.pt 0
pair_coeff      * * C H O N

timestep        0.0001
thermo          100

fix             1 all nvt temp 300 300 100
run             1000
```

- The `0` after the model filename is required for the unified ML-IAP interface.
- The element list after `* *` must be a subset of elements the model was trained on.

---

## 7. Running on THEMMES

```bash
# Single GPU
lmp -k on g 1 -sf kk -pk kokkos newton on neigh half -in input.in

```

---

## 8. Debugging Environment Variables

| Variable | Effect |
|---|---|
| `MACE_TIME=true` | Print timing info per step |
| `MACE_PROFILE=true` | Enable profiling |
| `MACE_ALLOW_CPU=true` | Allow CPU fallback (slow) |
| `MACE_FORCE_CPU=true` | Force CPU calculation |

---

## 9. Known Limitations

- ML-IAP plugin only works with **Kokkos on GPU**.
- **Multiple model heads** are not supported.
- Beta status — validate against standard MACE calculations.

---

## References

- MACE ML-IAP docs: https://mace-docs.readthedocs.io/en/latest/guide/lammps_mliap.html
- LAMMPS Kokkos build: https://docs.lammps.org/Build_extras.html#kokkos
- LAMMPS `pair_style mliap`: https://docs.lammps.org/pair_mliap.html

---

*Last updated: April 2026. THEMMES-specific (A100, CUDA 12.9).*
