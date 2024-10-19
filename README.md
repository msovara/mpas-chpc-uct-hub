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
