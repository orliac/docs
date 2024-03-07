# Installing BIPP on izar

## 1. Activate your VPN if not on EPFL campus

## 2. Log into izar.epfl.ch
```
$ ssh -Y <gaspar_username>@izar.epfl.ch
```

## 3. Open an interactive session (asking for 10+ CPU cores)
```
$ Sinteract -t 00-02:0:0 -c 20 -m 80G -g gpu:1 -p gpu -q gpu
```

## 4. If necessary, create a directory where to install BIPP and its dependencies
```
$ mkdir /path/to/my/soft    # For example: $ mkdir ~/Bluebild
```

:::danger
We'll refer to this installation directory as $INST_DIR hereinafter.
:::

## 5. Move into $INST_DIR
```
$ cd $INST_DIR
```

## 6. Install branch v2.1.0 of`finufft`
```
$ git clone -b v2.1.0 https://github.com/flatironinstitute/finufft
$ cd finufft/
$ module load gcc fftw/3.3.10-openmp
```
Create a file `make.inc` containing:
```
CXXFLAGS = -I${FFTW_ROOT}/include
LDFLAGS = -L${FFTW_ROOT}/lib
```
Then compile:
```
$ make clean && make test -j
```
At the end you should get something like:
```
check_finufft.sh single-precision done. Summary:
0 segfaults out of 8 tests done
0 fails out of 8 tests done
```

## 7. Install branch t3_d3 of `cufinufft` from Simon Frasch's fork
```
$ cd $INST_DIR
$ git clone -b t3_d3 https://github.com/AdhocMan/cufinufft.git
$ cd cufinufft
$ module load cuda
$ make clean && make -j
```

## 8. Create a Python virtual environment and activate it
:::info
See https://scitas-doc.epfl.ch/user-guide/software/python/python-venv/
:::
```
$ module load python
$ python -m venv --system-site-packages VENV
$ source VENV/bin/activate
```
:::success
Your command line should now look like:
```(VENV) [orliac@i63 Bluebild]$ ```
But for simplicity it will be indicated as `(VENV) $` hereafter.
:::

## 9. Install BIPP
```
(VENV) $ cd $INST_DIR
(VENV) $ git clone https://github.com/epfl-radio-astro/bipp.git
(VENV) $ cd bipp
(VENV) $ module load cmake openblas/0.3.20-openmp
(VENV) $ export FINUFFT_ROOT=${INST_DIR}/finufft
(VENV) $ export CUFINUFFT_ROOT=${INST_DIR}/cufinufft
(VENV) $ BIPP_GPU=CUDA CMAKE_PREFIX_PATH="${FINUFFT_ROOT};${CUFINUFFT_ROOT};${CMAKE_PREFIX_PATH}" python -m pip install .
```

## 10. Modify your $LD_LIBRARY_PATH
```
export LD_LIBRARY_PATH=${FINUFFT_ROOT}/lib:${CUFINUFFT_ROOT}/lib:${LD_LIBRARY_PATH}
```
:::info
To make this change permanent, you can modify your ```.bashrc``` file. Then log out and re-log in.
:::

## 11. Test your installation
If your installation is OK and ``$LD_LIBRARY_PATH`` is properly set then:
```
(VENV) $ python -c "import bipp"
```
should return just nothing.

## 12. Testing BIPP
First: log out and re-log in.
```
$ cd $INST_DIR
$ source VENV/bin/activate
(VENV) $ python -c "import bipp"
(VENV) $ cd bipp/examples/simulation/
(VENV) $ python lofar_bootes_ss.py 
```
