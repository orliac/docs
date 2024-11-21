# Installing BIPP on Kuma

## Connect to Kuma and launch an interactive session

```
$ ssh <gaspar_username>@kuma.hpc.epfl.ch
```

```
$ Sinteract -p h100 -g gpu:1 -t 03:00:00 -m 80G -c 16
```

```
$ mkdir -pv install_directory
```
and/or 

```
$ cd  install_directory
```


## Install dependency finufft (branch v2.1.0)

```
$ git clone -b v2.1.0 https://github.com/flatironinstitute/finufft
$ cd finufft/
```

Load necessary modules:
```
$ module purge
$ module load gcc
$ module load fftw/3.3.10-openmp
$ module load openblas
$ module load openmpi
$ module load cuda
$ module load python
$ module load tk
```

Create a `make.inc` file with:
```
$ printf "CXXFLAGS = -I${FFTW_ROOT}/include\nLDFLAGS = -L${FFTW_ROOT}/lib\n" > make.inc
```
The file should look like:
```
$ cat make.inc
CXXFLAGS = -I/ssoft/spack/pinot-noir/kuma-h100/v1/spack/opt/spack/linux-rhel9-zen4/gcc-13.2.0/fftw-3.3.10-f3727w6li4pbzd6o72ipeadnwkxdcrvu/include
LDFLAGS = -L/ssoft/spack/pinot-noir/kuma-h100/v1/spack/opt/spack/linux-rhel9-zen4/gcc-13.2.0/fftw-3.3.10-f3727w6li4pbzd6o72ipeadnwkxdcrvu/lib
```
NOTE: the fftw module needs to be loaded before generating the `make.inc` file.

Then compile:
```
$ make clean && make test -j
```
Check the log for correctness:
```
check_finufft.sh double-precision done. Summary:
0 segfaults out of 8 tests done
0 fails out of 8 tests done
...
check_finufft.sh single-precision done. Summary:
0 segfaults out of 8 tests done
0 fails out of 8 tests done
```

## Install dependency branch t3_d3 of `cufinufft` from Simon Frasch's fork
```
$ cd  install_directory
$ git clone -b t3_d3 https://github.com/AdhocMan/cufinufft.git
$ cd cufinufft
$ export NVARCH="-gencode arch=compute_80,code=sm_80 -gencode arch=compute_90,code=sm_90 -gencode arch=compute_90,code=compute_90"
$ make clean && make -j
```

## Create a Python virtual environment and activate it
```
$ python -m venv --system-site-packages VENV
$ source VENV/bin/activate
```

## Install BIPP

First, edit your `~/.bashrc` file to add the absolute paths to `finufft` and `cufinufft` that you installed above and add the following 3 lines:
```
export FINUFFT_ROOT=<install_directory>/finufft
export CUFINUFFT_ROOT=<install_directory>/cufinufft
export LD_LIBRARY_PATH=${FINUFFT_ROOT}/lib:${CUFINUFFT_ROOT}/lib:${LD_LIBRARY_PATH}
```
Then source the `~/.bashrc` file:
```
(VENV) $ source ~/.bashrc
```
And check the values of `$FINUFFT_ROOT` and `$CUFINUFFT_ROOT`.

```
(VENV) $ cd  install_directory
(VENV) $ git clone git@github.com:epfl-radio-astro/bipp.git
(VENV) $ cd bipp
(VENV) $ export CMAKE_PREFIX_PATH=$FINUFFT_ROOT:$CUFINUFFT_ROOT:$CMAKE_PREFIX_PATH
(VENV) $ BIPP_MPI=ON BIPP_GPU=CUDA python -m pip install .
```

## Test BIPP
```
(VENV) $ python -c "import bipp"
```
Should spit nothing.

## Run the distributed examples
```
(VENV) $ cd install_directory
(VENV) $ python bipp/examples/simulation/lofar_bootes_nufft.py
```

NOTE: you have to switch to a non-interactive backend (e.g. `agg`) and disbale image displays in your scripts.

## Running batch jobs (Slurm)

Create a submission script (e.g. `submit_bipp.sh`) along these lines:

```
#!/bin/bash -l

#SBATCH --partition h100
#SBATCH --ntasks 2
#SBATCH --gpus-per-task 1
#SBATCH --cpus-per-task 16
#SBATCH --mem 40G
#SBATCH --time 00-00:15:00

set -e

env | grep SLURM || true

module purge
module load gcc
module load fftw/3.3.10-openmp
module load openblas
module load openmpi
module load cuda
module load python
module load tk

### Edit here accordingly (installation path)
install_directory=/path/to/your/install_directory
[ -d ${install_directory} ] || (echo "${install_directory} not found." && exit 1)

### Edit here accordingly (Python virtual environment name)
source ${install_directory}VENV/bin/activate

time srun python ${install_directory}/bipp/examples/simulation/lofar_bootes_nufft.py

```
then submit with:
```
$ sbatch subimt_bipp.sh
```