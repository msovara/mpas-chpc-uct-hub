# 1. Required Dependencies
NetCDF requires:

- **zlib**: For data compression.
- **HDF5**: For hierarchical data storage (required for NetCDF-4).
- **Curl**: For remote data access (optional but recommended).
- **C and Fortran compilers**: For building the libraries (e.g., gcc, gfortran, or equivalent MPI compilers).

# 2. Download the Dependencies and Source Code
Get the source code for ```NetCDF-C``` and optionally ```NetCDF-Fortran``` from the official website:
```
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.9.2.tar.gz
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.6.0.tar.gz
```
Extract the archives:
```
tar -xvzf v4.9.2.tar.gz
tar -xvzf v4.6.0.tar.gz
```
# 3. Build and Install NetCDF-C
Navigate to the extracted directory and configure the build system:
```
cd netcdf-c-4.9.2
./configure --prefix=/path/to/install \
             --enable-netcdf-4 \
             --enable-dap \
             --with-hdf5=/path/to/hdf5 \
             --with-zlib=/path/to/zlib


```

## Build these from source or install using package manager. On LENGAU, build using Intel MPI
```
#!/bin/bash

module load chpc/parallel_studio_xe/18.0.2/2018.2.046
source /apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/bin64/mpivars.sh

export MPASDIR=/home/apps/chpc/earthMPAS-A-8.2.2-impi-pnc-pio-tau
export DIR=$MPASDIR/LIBRARIES
export CC=icc
export CXX=icc
export FC=ifort
export FCFLAGS="-m64 -I$DIR/netcdf/include  -I$DIR/grib2/include"
export F77=ifort
export FFLAGS=$FCFLAGS
export CFLAGS=$FCFLAGS
export PKG_CONFIG_PATH=$DIR/grib2/lib/pkgconfig:$DIR/netcdf/lib/pkgconfig:$DIR/pnetcdf/lib/pkgconfig
export MPICC=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiicc
export MPIF90=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort
export MPIF77=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort
export PATH=$DIR/netcdf/bin:$DIR/pnetcdf/bin:$DIR/grib2/bin:$PATH
export LD_LIBRARY_PATH=$DIR/grib2/lib:/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries_2018.2.199/linux/compiler/lib/intel64_lin:$DIR/netcdf/lib:$DIR/pnetcdf/lib:$LD_LIBRARY_PATH
export NETCDF4=1
export NETCDF=$DIR/netcdf
export HDF5=$DIR/grib2
export LDFLAGS="-L$DIR/grib2/lib"
export CPPFLAGS=-I$DIR/grib2/include
export JASPERLIB=$DIR/grib2/lib
export JASPERINC=$DIR/grib2/include
export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export PATH=$WRFDIR/WPS:$WRFDIR/WPS/util:$PATH
export PATH=$WRFDIR/WRF/run:$PATH
export NCARG_ROOT=$WRFDIR/ncl
export PATH=$PATH:$NCARG_ROOT/bin
export GADDIR=/apps/chpc/earth/grads-2.1.a3
export PATH=$PATH:$GADDIR/bin
export PATH=$PATH:$WRFDIR/ARWpost
export PATH=$PATH:$WRFDIR/UPPV3.1.1/bin
ulimit -s unlimited


```




# MPAS

Set up a ```tmux or screen``` session and then start the download process by editing dates in the forcing files: 
```cfsr_dwn_prslev_modified.py ```& ```cfsr_dwn_sfc_modified.py```. This assumes that you already have an account on the cfsr website. 
```
passwd = ""
email = 'mvsovara@gmail.com'
dsnum = 'd094000'
startdate = '199701010000'
enddate = '200001150600'
oformat = "WMO_GRIB1"
product = "6-hour Forecast"
grid = "720:361:90N:0E:90S:359.5E:0.5:0.5"
filename = "cfsr_prs_" + startdate + "-" + enddate + oformat
```

Then edit the date lines and switches accordingly in the automation script ```bash_auto_run ```
```
#!/bin/bash

# ==== Forcing dataset section ===============

dtpath="data/cfsr"                              ## where do you want all dataset to be downloaded to?
prsdwnstartdatetime=199701010000                ## start datetime for downloading pressure level forcing data
prsdwnenddatetime=200001150000                  ## Note: at least more than 6 days ahead of prsdwnstartdatetime, to avoid ungrib error
sfcdwnstartdatetime=199701010000                ## Start datetime for downloading surface forcing dataset
sfcdwnenddatetime=200001150000                  ## End datatime for downloading surface forcing dataset
singlemultiyears=1                              ## Option 0 - download forcing data year by year so that each year have its own directory. It is going to be a short year-to-year simulation
                                                ## Option 1 - download forcing data for all the years so that all the years reside in just one directory. It will be used for long/multi-year simulation


# ===== Ungrib section ========================

prsungrbstartdatetime=1997-01-01_00:00:00       ## Note: make sure its thesame as 'prsdwnstartdatetime' and formatted as it is (using '_' and ':" as date and time separator) to avoid ungrib error
prsungrbenddatetime=2000-01-15_06:00:00         ## Note: at most the end datetime should be 18 hrs less than 'prsdwnenddatetime'
sfcungrbstartdatetime=1997-01-01_00:00:00       ## Note: Should be less than or equal to 'prsungrbstartdatetime'
sfcungrbenddatetime=2000-01-15_00:00:00         ## Note: at most, it should be less than or equal to 'sfcdwnenddatetime'
manyyearsungrib=1                               ## Option 0 - Ungrib forcing data year by year.
                                                ## Option 1 - Ungrib forcing data all at once, starting from 'sfcungrbstartdatetime' to 'sfcungrbenddatetime'

# ========= Simulation settings section ========
simstartdate=1997-01-01_06:00:00                ## Note: Should be thesame as the 'prsungrbstartdatetime'
simenddate=2000-01-15_00:00:00                  ## Can be any datetime you want the simulation to stop

rundir="60km_uniform"                   ## Name of the model directory to create or over write. Note model excutables, files and output will be copied to this directory
rsltn="60km"                                    ## What do resolution do you want to run? Remember to qoute it as string and include the unit
lowRsltn=60                                     ## Note: the value is qouted (i.e. not "" or ''). For variable resolution, set to the lowest resolution. For uniform resolution, use thesame value as 'rsltn'
meshdir="meshes/60km"                           ## Where is location of the mesh for the particular resolutin you wish to run?
ncores=192                                      ## How many processors do which to use. Note: make sure a partitioning file for the number processors you have choosen is avaialable in the 'meshdir'
LOGFILE="log.auto_60km"         ## This bash script generates a log file. What do you wish to name the logfile?

# ======== Simulation Switches =============
# Option 0 - do not skip, run the process
# Option 1 - do not run, skip the process

skipdwn=0
skipung=0
skipinit=0
skipatmos=0

```

Move to edit the files ```bash_ung.sh```, ```bash_dwn.sh```, ```bash_init.sh```, ```run_mpas_auto_init.qsub```, ```run_ungrib.qsub```, ```run_mpas_atmos.qsub``` accordingly and the source the set script ```setMPAS_env.sh``` before running the automation script ```bash_auto_run```

Data download token: https://rda.ucar.edu/accounts/profile/
