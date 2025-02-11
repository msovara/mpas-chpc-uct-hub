# Developer Notes

## 1. Required Dependencies required for building MPAS version 8.2 (Order of Build): 

### Basic Build Dependencies
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

# Function to clean previous builds
clean_builds() {
    echo "Cleaning previous builds..."
    # Clean installation and build directories
    rm -rf $INSTALL_DIR/* $BUILD_DIR/*
    # Clean source directories
    cd $SOURCE_DIR || exit 1
    rm -rf zlib-1.3.1 curl-8.5.0 mpich-4.2.2 hdf5-1.14.3 \
        netcdf-c-4.9.2 netcdf-fortran-4.6.1 pnetcdf-1.12.3 MPAS-Model
    # Clean downloaded tarballs
    rm -f zlib-1.3.1.tar.gz curl-8.5.0.tar.gz mpich-4.2.2.tar.gz \
        hdf5-1.14.3.tar.gz netcdf-c-4.9.2.tar.gz netcdf-fortran-4.6.1.tar.gz \
        pnetcdf-1.12.3.tar.gz
    echo "Clean completed."
}

# Base directories
export BASE_DIR=/home/msovara/lustre/SoftwareBuilds/mpas-gcc-build
export INSTALL_DIR=$BASE_DIR/install  # Where dependencies will be installed
export BUILD_DIR=$BASE_DIR/build      # Temporary build directory
export SOURCE_DIR=$BASE_DIR/sources   # Source code for dependencies
export MPAS_DIR=$SOURCE_DIR/MPAS-Model  # MPAS source code directory

# Define compiler paths
export GCC_DIR="/apps/chpc/compmech/spack/opt/spack/linux-centos7-haswell/gcc-9.2.0/gcc-12.1.0-zhlzestmsfeddbgqfhyj7apvzanr4otx"

# Ask user if they want to clean previous builds
read -p "Do you want to clean previous builds? (y/n) " answer
case ${answer:0:1} in
    y|Y )
        clean_builds  # Clean previous builds if user confirms
    ;;
    * )
        echo "Skipping clean..."  # Skip cleaning if user declines
    ;;
esac

# Create directories
echo "Creating directories..."
mkdir -p $INSTALL_DIR $BUILD_DIR $SOURCE_DIR
cd $BUILD_DIR

# Reset PATH and LD_LIBRARY_PATH to system defaults first
export PATH=/usr/local/bin:/usr/bin:/bin
export LD_LIBRARY_PATH=/lib64:/lib

# Load base compiler and clear environment
module purge  # Unload all loaded modules
conda deactivate 2>/dev/null  # Deactivate conda if active

# Load GCC module
module load chpc/compmech/gcc/12.1.0  # Load GCC 12.1.0
module load chpc/git/2.41.0  # Load Git for cloning repositories

# Set compiler environment
export CC=$GCC_DIR/bin/gcc  # C compiler
export CXX=$GCC_DIR/bin/g++  # C++ compiler
export FC=$GCC_DIR/bin/gfortran  # Fortran compiler
export F77=$GCC_DIR/bin/gfortran  # Fortran 77 compiler
export F90=$GCC_DIR/bin/gfortran  # Fortran 90 compiler

# Set paths (without duplicates)
export PATH=$INSTALL_DIR/bin:$GCC_DIR/bin:$PATH  # Add installation and GCC paths to PATH
export LD_LIBRARY_PATH=$INSTALL_DIR/lib:$INSTALL_DIR/lib64:$GCC_DIR/lib64:$GCC_DIR/lib:$LD_LIBRARY_PATH  # Add libraries to LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$INSTALL_DIR/lib/pkgconfig:$PKG_CONFIG_PATH  # Add pkg-config path

# Verify environment
echo "Verifying environment:"
echo "PATH=$PATH"
echo "LD_LIBRARY_PATH=$LD_LIBRARY_PATH"
echo "Testing GCC:"
$CC --version  # Verify GCC version

# --------------------------
# Build Basic Dependencies
# --------------------------
# Build zlib (compression library)
cd $SOURCE_DIR
if [ ! -f zlib-1.3.1.tar.gz ]; then
    wget https://zlib.net/zlib-1.3.1.tar.gz  # Download zlib
    tar xzf zlib-1.3.1.tar.gz  # Extract zlib
fi
cd zlib-1.3.1
./configure --prefix=$INSTALL_DIR  # Configure zlib
make -j8 && make install  # Build and install zlib

# Build curl (for remote data access)
cd $SOURCE_DIR
if [ ! -f curl-8.5.0.tar.gz ]; then
    wget https://curl.se/download/curl-8.5.0.tar.gz  # Download curl
    tar xzf curl-8.5.0.tar.gz  # Extract curl
fi
cd curl-8.5.0
make distclean || true  # Clean previous builds
rm -f config.cache  # Remove cached configuration

# Set up environment for curl
export CPPFLAGS="-I$INSTALL_DIR/include"  # Include paths
export CFLAGS="-fPIC -O2"  # Compiler flags
export LDFLAGS="-L$INSTALL_DIR/lib -L/usr/lib64"  # Linker flags

# Configure curl with SSL support
./configure --prefix=$INSTALL_DIR \
    --with-zlib=$INSTALL_DIR \
    --with-ssl=/usr \
    --enable-ipv6 \
    --enable-unix-sockets

make -j8 && make install  # Build and install curl

# --------------------------
# Build MPI Stack
# --------------------------
# Build MPICH (MPI implementation)
cd $SOURCE_DIR
if [ ! -f mpich-4.2.2.tar.gz ]; then
    wget https://www.mpich.org/static/downloads/4.2.2/mpich-4.2.2.tar.gz  # Download MPICH
    tar xzf mpich-4.2.2.tar.gz  # Extract MPICH
fi
cd mpich-4.2.2
make clean  # Clean previous builds
./configure --prefix=$INSTALL_DIR \
    --enable-shared \
    --enable-fortran \
    --with-device=ch3:nemesis  # Configure MPICH
make -j8 && make install  # Build and install MPICH

# Update PATH after MPICH installation
export PATH=$INSTALL_DIR/bin:$PATH  # Add MPICH binaries to PATH
export LD_LIBRARY_PATH=$INSTALL_DIR/lib:$LD_LIBRARY_PATH  # Add MPICH libraries to LD_LIBRARY_PATH

# --------------------------
# Build Parallel I/O Stack
# --------------------------
# Build HDF5 (hierarchical data format)
cd $SOURCE_DIR
if [ ! -f hdf5-1.14.3.tar.gz ]; then
    wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.3/src/hdf5-1.14.3.tar.gz  # Download HDF5
    tar xzf hdf5-1.14.3.tar.gz  # Extract HDF5
fi
cd hdf5-1.14.3
make distclean || true  # Clean previous builds
./configure --prefix=$INSTALL_DIR \
    --enable-parallel \
    --enable-fortran \
    --enable-hl \
    --with-zlib=$INSTALL_DIR \
   CC=mpicc \
   CXX=mpicxx \
   FC=mpif90 \
   CFLAGS="-fPIC -O2" \
   CPPFLAGS="-I$INSTALL_DIR/include" \
   LDFLAGS="-L$INSTALL_DIR/lib"  # Configure HDF5 with parallel support
make -j8 && make install  # Build and install HDF5

# Verify HDF5 build
echo "Checking HDF5 parallel support:"
h5pcc -showconfig | grep "Parallel HDF5"  # Verify parallel HDF5 support

# Build NetCDF-C (network Common Data Form)
cd $SOURCE_DIR
if [ ! -f netcdf-c-4.9.2.tar.gz ]; then
    wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.9.2.tar.gz -O netcdf-c-4.9.2.tar.gz  # Download NetCDF-C
    tar xzf netcdf-c-4.9.2.tar.gz  # Extract NetCDF-C
fi
export HDF5_DIR=$INSTALL_DIR
export CC=mpicc
export CXX=mpicxx
export FC=mpif90
export CPPFLAGS="-I$INSTALL_DIR/include"
export CFLAGS="-fPIC -O2"
export LDFLAGS="-L$INSTALL_DIR/lib"

cd netcdf-c-4.9.2
make distclean || true  # Clean previous builds
./configure --prefix=$INSTALL_DIR \
    --enable-parallel4 \
    --enable-shared \
    --disable-dap \
    --with-hdf5=$HDF5_DIR  # Configure NetCDF-C
make -j8 && make install  # Build and install NetCDF-C

# Build NetCDF-Fortran
cd $SOURCE_DIR
if [ ! -f netcdf-fortran-4.6.1.tar.gz ]; then
    wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.6.1.tar.gz -O netcdf-fortran-4.6.1.tar.gz  # Download NetCDF-Fortran
    tar xzf netcdf-fortran-4.6.1.tar.gz  # Extract NetCDF-Fortran
fi
cd netcdf-fortran-4.6.1
make distclean || true  # Clean previous builds
./configure --prefix=$INSTALL_DIR --enable-shared  # Configure NetCDF-Fortran
make -j8 && make install  # Build and install NetCDF-Fortran

# Build PNetCDF (Parallel NetCDF)
cd $SOURCE_DIR
if [ ! -f pnetcdf-1.12.3.tar.gz ]; then
    wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz  # Download PNetCDF
    tar xzf pnetcdf-1.12.3.tar.gz  # Extract PNetCDF
fi
cd pnetcdf-1.12.3
./configure --prefix=$INSTALL_DIR  # Configure PNetCDF
make -j8 && make install  # Build and install PNetCDF

# --------------------------
# Build MPAS
# --------------------------
# Clone MPAS if needed
cd $SOURCE_DIR
if [ ! -d "MPAS-Model" ]; then
    echo "Cloning MPAS-Model..."
    git clone https://github.com/MPAS-Dev/MPAS-Model.git || \
    git clone git://github.com/MPAS-Dev/MPAS-Model.git  # Clone MPAS repository
    # Apply compatibility patch
    cd MPAS-Model
    echo "Patching for git compatibility..."
    find src -type f -exec sed -i -E 's/git -C ([^ ]+) /(cd \1 \&\& git) /g' {} +  # Patch for Git compatibility
    cd ..
fi
cd MPAS-Model
git checkout master
git pull  # Update to the latest version

# Set up MPI environment with GCC paths
export PATH=$INSTALL_DIR/bin:$GCC_DIR/bin:$PATH
export LD_LIBRARY_PATH=$INSTALL_DIR/lib:$INSTALL_DIR/lib64:$GCC_DIR/lib64:$GCC_DIR/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$INSTALL_DIR/lib/pkgconfig:$PKG_CONFIG_PATH

# Set compiler environment explicitly for GCC + MPI
export CC="mpicc -cc=$GCC_DIR/bin/gcc"
export FC="mpif90 -fc=$GCC_DIR/bin/gfortran"
export CXX="mpicxx -cxx=$GCC_DIR/bin/g++"
export NETCDF=$INSTALL_DIR
export PNETCDF=$INSTALL_DIR

# Set dependencies (PIO installed via Spack)
export NETCDF=$INSTALL_DIR
export PNETCDF=$INSTALL_DIR
export PIO=$(spack location -i parallelio)
export PATH=$PIO/bin:$NETCDF/bin:$PNETCDF/bin:$PATH
export LD_LIBRARY_PATH=$PIO/lib:$NETCDF/lib:$PNETCDF/lib:$LD_LIBRARY_PATH

# Add include paths with GCC-specific directories
export CPPFLAGS="-I$INSTALL_DIR/include"
export CFLAGS="-fPIC -O2"
export FFLAGS="-fPIC -O2 -ffree-line-length-none -fconvert=big-endian -ffree-form -fdefault-real-8"
export LIBS="-lnetcdff -lnetcdf -lpnetcdf -lhdf5_hl -lhdf5 -lcurl -lz"

# Clean previous builds
make clean CORE=atmosphere
make clean CORE=init_atmosphere
rm -rf *.o *.mod *.a

# Build atmosphere core
make gfortran CORE=atmosphere \
    PRECISION=single \
    USE_PIO2=false \
    DEBUG=false \
    OPENMP=false \
    AUTOCLEAN=true \
    -j8  # Build MPAS atmosphere core

# Verify final build
if [ -f atmosphere_model ]; then
    echo "MPAS build successful!"
    ldd atmosphere_model  # Verify linked libraries
    mkdir -p $INSTALL_DIR/bin
    cp atmosphere_model $INSTALL_DIR/bin/  # Copy executable to installation directory
else
    echo "MPAS build failed!"
    exit 1
fi

echo "Full build completed successfully!"


```


Contact information: msovara@csir.co.za

Data download token: https://rda.ucar.edu/accounts/profile/
