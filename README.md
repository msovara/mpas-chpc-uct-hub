# Developer Notes

## 1. Dependencies required for building MPAS version 8.2: 

### Build Basic Dependencies
1. **Zlib**: A software library used for data compression (often needed early in builds to handle compressed data during the build process).
2. **Curl**: A tool and library for transferring data with URLs, used for accessing remote resources over HTTP, FTP, and other protocols (optional but recommended for accessing remote data).

### Build MPI Stack
3. **MPICH**: An implementation of the MPI (Message Passing Interface) standard for parallel computing, enabling communication between processes in a distributed system (required for building parallel applications).

### Build Parallel I/O Stack
4. **HDF5**: A file format and set of tools for managing hierarchical data, often used for storing large amounts of scientific data (required for NetCDF-4, which is used for climate and weather data storage).
5. **C and Fortran Compilers**: Essential for compiling the source code of libraries and applications (e.g., GCC for C/C++ and GFortran for Fortran, necessary for building scientific software and libraries, including those needed for MPI support).
6. **PNETCDF**: A parallel I/O library designed to handle large scientific datasets in parallel computing environments (requires MPI support). 
4. **Parallel IO**: A method for efficient input/output (I/O) operations in parallel computing, allowing multiple processes to read/write data simultaneously (typically integrated after basic libraries are in place).

## 2. Script to build dependencies and compile MPAS v8.2 with TAU Hooks

The MPAS code is distributed directly from the GitHub repository where it is developed. While it's possible to navigate to [MPAS-Dev/MPAS-Model](https://github.com/MPAS-Dev/MPAS-Model) and obtain the code by clicking on the "Releases" tab, it's much faster to clone the repository directly on the command-line. Run the code below to build the MPAS dependencies and compile MPAS.

```bash
#!/bin/bash

# Exit on error and enable error trapping
set -e
trap 'echo "Error on line $LINENO"' ERR

# Function to clean previous builds
clean_builds() {
    echo "Cleaning previous builds..."
    rm -rf $INSTALL_DIR/* $BUILD_DIR/*
    cd $SOURCE_DIR || exit 1
    rm -rf zlib-1.3.1 curl-8.5.0 mpich-4.2.2 hdf5-1.14.3 \
        netcdf-c-4.9.2 netcdf-fortran-4.6.1 pnetcdf-1.12.3 MPAS-Model tau-2.33
    rm -f zlib-1.3.1.tar.gz curl-8.5.0.tar.gz mpich-4.2.2.tar.gz \
        hdf5-1.14.3.tar.gz netcdf-c-4.9.2.tar.gz netcdf-fortran-4.6.1.tar.gz \
        pnetcdf-1.12.3.tar.gz tau.tgz
    echo "Clean completed."
}

# Function to verify environment
verify_environment() {
    echo "Verifying environment setup..."
    if [ ! -d "$GCC_DIR" ]; then
        echo "GCC directory not found: $GCC_DIR"
        exit 1
    fi
    echo "Environment verification completed."
}

# Base directories
export BASE_DIR=/home/msovara/lustre/SoftwareBuilds/mpas-gcc-build
export INSTALL_DIR=$BASE_DIR/install
export BUILD_DIR=$BASE_DIR/build
export SOURCE_DIR=$BASE_DIR/sources
export MPAS_DIR=$SOURCE_DIR/MPAS-Model

# Define compiler paths
export GCC_DIR="/apps/chpc/compmech/spack/opt/spack/linux-centos7-haswell/gcc-9.2.0/gcc-12.1.0-zhlzestmsfeddbgqfhyj7apvzanr4otx"

# Ask user if they want to clean previous builds
read -p "Do you want to clean previous builds? (y/n) " answer
case "$answer" in
    y|Y )
        echo "Cleaning previous builds..."
        # Add your cleanup command here
        ;;
    n|N )
        echo "Skipping clean..."
        ;;
    * )
        echo "Invalid input. Please enter y or n."
        ;;
esac

# Create directories
echo "Creating directories..."
mkdir -p $INSTALL_DIR $BUILD_DIR $SOURCE_DIR
cd $BUILD_DIR

# Reset environment
export PATH=/usr/local/bin:/usr/bin:/bin
export LD_LIBRARY_PATH=/lib64:/lib

# Load modules
module purge
source /apps/chpc/chem/anaconda3-2021.11/etc/profile.d/conda.sh # Source conda
conda deactivate 2>/dev/null
module load chpc/compmech/gcc/12.1.0
module load chpc/git/2.41.0

# Initial compiler setup (using GCC directly)
export CC=$GCC_DIR/bin/gcc
export CXX=$GCC_DIR/bin/g++
export FC=$GCC_DIR/bin/gfortran
export F77=$GCC_DIR/bin/gfortran
export F90=$GCC_DIR/bin/gfortran

# Set paths
export PATH=$INSTALL_DIR/bin:$GCC_DIR/bin:$PATH
export LD_LIBRARY_PATH=$INSTALL_DIR/lib:$INSTALL_DIR/lib64:$GCC_DIR/lib64:$GCC_DIR/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$INSTALL_DIR/lib/pkgconfig:$PKG_CONFIG_PATH

# Build zlib
cd $SOURCE_DIR
if [ ! -f zlib-1.3.1.tar.gz ]; then
    wget https://zlib.net/zlib-1.3.1.tar.gz
    tar xzf zlib-1.3.1.tar.gz
fi
cd zlib-1.3.1
./configure --prefix=$INSTALL_DIR
make -j8 && make install

# Build curl
cd $SOURCE_DIR
if [ ! -f curl-8.5.0.tar.gz ]; then
    wget https://curl.se/download/curl-8.5.0.tar.gz
    tar xzf curl-8.5.0.tar.gz
fi
cd curl-8.5.0
make distclean || true
rm -f config.cache
export CPPFLAGS="-I$INSTALL_DIR/include"
export CFLAGS="-fPIC -O2"
export LDFLAGS="-L$INSTALL_DIR/lib -L/usr/lib64"
./configure --prefix=$INSTALL_DIR \
    --with-zlib=$INSTALL_DIR \
    --with-ssl=/usr \
    --enable-ipv6 \
    --enable-unix-sockets
make -j8 && make install

# Build MPICH (without TAU)
cd $SOURCE_DIR
if [ ! -f mpich-4.2.2.tar.gz ]; then
    wget https://www.mpich.org/static/downloads/4.2.2/mpich-4.2.2.tar.gz
    tar xzf mpich-4.2.2.tar.gz
fi
cd mpich-4.2.2
make clean || true
./configure --prefix=$INSTALL_DIR \
    --enable-shared \
    --enable-fortran \
    --with-device=ch3:nemesis
make -j8 && make install

# Update PATH and LD_LIBRARY_PATH after MPICH installation
export PATH=$INSTALL_DIR/bin:$PATH
export LD_LIBRARY_PATH=$INSTALL_DIR/lib:$LD_LIBRARY_PATH
INSTALL_DIR=/home/msovara/lustre/SoftwareBuilds/mpas-gcc-build/install

# Build TAU from source
cd $SOURCE_DIR
if [ ! -f tau.tgz ]; then
    wget http://tau.uoregon.edu/tau.tgz
    tar xzf tau.tgz
fi
cd tau-2.34
make clean
./configure -prefix=$INSTALL_DIR/tau \
    -mpi \
    -mpiinc=$INSTALL_DIR/include \
    -mpilib=$INSTALL_DIR/lib \
    -cc=gcc \
    -fortran=gfortran \
    -iowrapper

# Set up TAU environment
export TAU_DIR=$INSTALL_DIR/tau
export TAU_MAKEFILE=$TAU_DIR/x86_64/lib/Makefile.tau-mpi
export TAU_OPTIONS="-optRevert"
export PATH=$TAU_DIR/x86_64/bin:$PATH
export LD_LIBRARY_PATH=$TAU_DIR/x86_64/lib:$LD_LIBRARY_PATH

# Set TAU compiler wrappers to use the built MPICH
export CC="$TAU_DIR/x86_64/bin/tau_cc.sh -cc=$INSTALL_DIR/bin/mpicc"
export CXX="$TAU_DIR/x86_64/bin/tau_cxx.sh -cxx=$INSTALL_DIR/bin/mpicxx"
export FC="$TAU_DIR/x86_64/bin/tau_f90.sh -fc=$INSTALL_DIR/bin/mpif90"
export F77="$TAU_DIR/x86_64/bin/tau_f77.sh -fc=$INSTALL_DIR/bin/mpif77"
export F90="$TAU_DIR/x86_64/bin/tau_f90.sh -fc=$INSTALL_DIR/bin/mpif90"

# Build HDF5 with TAU instrumentation
cd $SOURCE_DIR
if [ ! -f hdf5-1.14.3.tar.gz ]; then
    wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.3/src/hdf5-1.14.3.tar.gz
    tar xzf hdf5-1.14.3.tar.gz
fi
cd hdf5-1.14.3
make distclean || true
./configure --prefix=$INSTALL_DIR \
    --enable-parallel \
    --enable-fortran \
    --enable-hl \
    --with-zlib=$INSTALL_DIR \
    CFLAGS="-fPIC -O2" \
    CPPFLAGS="-I$INSTALL_DIR/include" \
    LDFLAGS="-L$INSTALL_DIR/lib"
make -j8 && make install

# Build NetCDF-C with TAU instrumentation
cd $SOURCE_DIR
if [ ! -f netcdf-c-4.9.2.tar.gz ]; then
    wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.9.2.tar.gz -O netcdf-c-4.9.2.tar.gz
    tar xzf netcdf-c-4.9.2.tar.gz
fi
export HDF5_DIR=$INSTALL_DIR
cd netcdf-c-4.9.2
make distclean || true
./configure --prefix=$INSTALL_DIR \
    --enable-parallel4 \
    --enable-shared \
    --disable-dap \
    --with-hdf5=$HDF5_DIR
make -j8 && make install

# Build NetCDF-Fortran with TAU instrumentation
cd $SOURCE_DIR
if [ ! -f netcdf-fortran-4.6.1.tar.gz ]; then
    wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.6.1.tar.gz -O netcdf-fortran-4.6.1.tar.gz
    tar xzf netcdf-fortran-4.6.1.tar.gz
fi
cd netcdf-fortran-4.6.1
make distclean || true
./configure --prefix=$INSTALL_DIR --enable-shared
make -j8 && make install

# Build PNetCDF with TAU instrumentation
cd $SOURCE_DIR
if [ ! -f pnetcdf-1.12.3.tar.gz ]; then
    wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz
    tar xzf pnetcdf-1.12.3.tar.gz
fi
cd pnetcdf-1.12.3
./configure --prefix=$INSTALL_DIR
make -j8 && make install

# Clone and build MPAS with TAU instrumentation
cd $SOURCE_DIR
if [ ! -d "MPAS-Model" ]; then
    echo "Cloning MPAS-Model..."
    git clone https://github.com/MPAS-Dev/MPAS-Model.git || \
    git clone git://github.com/MPAS-Dev/MPAS-Model.git
    cd MPAS-Model
    echo "Patching for git compatibility..."
    find src -type f -exec sed -i -E 's/git -C ([^ ]+) /(cd \1 \&\& git) /g' {} +
    cd ..
fi
cd MPAS-Model
git checkout master
git pull

# Set up environment for MPAS build
export NETCDF=$INSTALL_DIR
export PNETCDF=$INSTALL_DIR
export PIO=$(spack location -i parallelio)
export PATH=$PIO/bin:$NETCDF/bin:$PNETCDF/bin:$PATH
export LD_LIBRARY_PATH=$PIO/lib:$NETCDF/lib:$PNETCDF/lib:$LD_LIBRARY_PATH

# Enhanced compiler flags for TAU
export CPPFLAGS="-I$INSTALL_DIR/include -I$TAU_DIR/include"
export CFLAGS="-fPIC -O2 -g"
export FFLAGS="-fPIC -O2 -g -ffree-line-length-none -fconvert=big-endian -ffree-form -fdefault-real-8"
export LIBS="-ltau -lnetcdff -lnetcdf -lpnetcdf -lhdf5_hl -lhdf5 -lcurl -lz"

# Clean MPAS
make clean CORE=atmosphere
make clean CORE=init_atmosphere
rm -rf *.o *.mod *.a

# Build MPAS atmosphere core
make gfortran CORE=atmosphere \
    PRECISION=single \
    USE_PIO2=false \
    DEBUG=false \
    OPENMP=false \
    AUTOCLEAN=true \
    -j8

# Verify final build
if [ -f atmosphere_model ]; then
    echo "MPAS build successful with TAU instrumentation!"
    echo "Run with:"
    echo "mpirun -np <NPROCS> tau_exec ./atmosphere_model"
    ldd atmosphere_model
    mkdir -p $INSTALL_DIR/bin
    cp atmosphere_model $INSTALL_DIR/bin/
else
    echo "MPAS build failed!"
    exit 1
fi

echo "TAU-instrumented build completed successfully!"
```
---

The entire build process should take about 26 minutes and 27.694 seconds. 

Contact information: msovara@csir.co.za

## TAU and User Guide under development...

Data download token: https://rda.ucar.edu/accounts/profile/
