---
author: louis_saddier
comments: true
date: 2023-11-23 00:30:00+00:00
layout: post
slug: nemo4-PSMN-E5-01
title: Building and Running XIOS and NEMO4 on PSMN (ENS de Lyon HPC).
categories:
- nemo4
---
We take the first step of downloading, compiling and testing XIOS and NEMO4 code on **PSMN (ENS de Lyon; Lyon, France)**. This is a post based on an installation made by Cerasela Calugaru. Thank you Cerasela ! Contact: louis.saddier@ens-lyon.fr

Pre-requisites:

* [PSMN account](https://www.ens-lyon.fr/PSMN/)

Suggested reading:

* [PSMN documentation](https://www.ens-lyon.fr/PSMN/Documentation/)

* [XIOS documentation](https://forge.ipsl.jussieu.fr/ioserver/wiki/documentation)

* [NEMO4 short user guide](https://sites.nemo-ocean.io/user-guide/)

## Installing XIOS

Log-into PSMN into a partition node (e.g., Lake). Set-up your environment with:
```bash
module use /applis/PSMN/debian13/E5/modules/all/
module load netCDF-Fortran/4.6.1-gompi-2023a
```

Get XIOS :
```bash
svn co http://forge.ipsl.jussieu.fr/ioserver/svn/XIOS/trunk XIOS
```

Now we need to create `arch` files to indicate the options we want when we will compile XIOS. First we will create a `.path` file:
```bash
cd XIOS/arch/
vi arch-X64_PSMN.path
```
and write:
```bash
NETCDF_INCDIR="-I$NETCDFF_INCDIR -I$NETCDF_INC_DIR"
NETCDF_LIBDIR="-L$NECDFF_LIBDIR -L$NETCDF_LIB_DIR"
NETCDF_LIB="-lnetcdff -lnetcdf"

MPI_INCDIR=""
MPI_LIBDIR=""
MPI_LIB=""

HDF5_INCDIR="-I$HDF5_INC_DIR"
HDF5_LIBDIR="-L$HDF5_LIB_DIR"
HDF5_LIB="-lhdf5_hl -lhdf5 -lhdf5 -lz"

OASIS_INCDIR="-I$PWD/../../oasis3-mct/BLD/build/lib/psmile.MPI1"
OASIS_LIBDIR="-L$PWD/../../oasis3-mct/BLD/lib"
OASIS_LIB="-lpsmile.MPI1 -lscrip -lmct -lmpeu"
```

Then, we will create a `.fcm` file:
```bash
vi arch-X64_PSMN.fcm
```
and write:
```bash
%CCOMPILER      mpicc
%FCOMPILER      mpif90
%LINKER         mpif90

%BASE_CFLAGS    -w -std=c++11 -D__XIOS_EXCEPTION
%PROD_CFLAGS    -O3 -D BOOST_DISABLE_ASSERTS
%DEV_CFLAGS     -g -traceback
%DEBUG_CFLAGS   -DBZ_DEBUG -g -traceback -fno-inline

%BASE_FFLAGS    -D__NONE__
%PROD_FFLAGS    -O3
%DEV_FFLAGS     -g -O2 -traceback
%DEBUG_FFLAGS   -g -traceback

%BASE_INC       -D__NONE__
%BASE_LD        -lstdc++

%CPP            mpicc -EP
%FPP            cpp -P
%MAKE           gmake
```

Finally, we will create a `.env` file:
```bash
vi arch-X64_PSMN.env
```
and then write:
```bash
module purge
module use /applis/PSMN/debian13/E5/modules/all/
module load netCDF-Fortran/4.6.1-gompi-2023a
export HDF5_INC_DIR=$HDF5_DIR/include
export HDF5_LIB_DIR=$HDF5_DIR/lib
export NETCDFF_INCDIR=/applis/PSMN/debian13/E5/software/netCDF-Fortran/4.6.1-gompi-2023a/include
export NETCDFF_LIBDIR=/applis/PSMN/debian13/E5/software/netCDF-Fortran/4.6.1-gompi-2023a/lib
export NETCDF_INC_DIR=/applis/PSMN/debian13/E5/software/netCDF/4.9.2-gompi-2023a/include
export NETCDF_LIB_DIR=/applis/PSMN/debian13/E5/software/netCDF/4.9.2-gompi-2023a/lib
```

Now we will compile XIOS with the options we just created (parallelized here with 4 processors):
```bash
cd ..
./make_xios --arch X64_PSMN --job 4
```

To test XIOS:
```bash
cd generic_testcase
mpirun -np 4 ../bin/generic_testcase.exe
```

If no errors appear, XIOS should be well installed.

## Installing NEMO

Get NEMO4, in my case I'm using the 4.2.1 version:
```bash
git clone --branch 4.2.1 https://forge.nemo-ocean.eu/nemo/nemo.git nemo_4.2.1
cd nemo_4.2.1/
```

If not already done do:
```bash
module use /applis/PSMN/debian13/E5/modules/all/
module load netCDF-Fortran/4.6.1-gompi-2023a
```

Then:
```bash
source XIOS/bin/fcm_env.ksh
```

As in the installation of XIOS, we need to modifiate some `arch` files that contain Fortran options for the compilation of NEMO:
```bash
cd arch
cp arch-linux_gfortran.fcm arch-PSMN.fcm
vi arch-PSMN.fcm
```

Uncomment and modify these 4 lines:
> Note that you need to specify in the last line the path where you installed XIOS. In my case it was in my folder `/home/lsaddier/nemo/models`
```bash
%HDF5_HOME           $HDF5_DIR
%NCDF_C_HOME         /applis/PSMN/debian13/E5/software/netCDF/4.9.2-gompi-2023a
%NCDF_F_HOME         /applis/PSMN/debian13/E5/software/netCDF-Fortran/4.6.1-gompi-2023a
%XIOS_HOME           /home/lsaddier/nemo/models/XIOS
```

On the `%FCFLAGS` line, add a last argument:
```bash
%FCFLAGS             -fdefault-real-8 -O3 -funroll-all-loops -fcray-pointer -ffree-line-length-none -fallow-argument-mismatch
```

Then we are ready to compile NEMO. Let's try with the `GYRE_PISCES` configuration:
```bash
cd ..
./makenemo -m 'PSMN' -r 'GYRE_PISCES' -n 'MY_GYRE_PISCES'
```

Finally you can test the execution of NEMO (here with 4 processors):
```bash
cd /nemo_4.2.1/cfgs/MY_GYRE_PISCES/EXP00
mpirun -n 4 ./nemo
```
