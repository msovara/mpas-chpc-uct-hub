# MPAS

Start download process by editing dates in the forcing files: 
```cfsr_dwn_prslev_modified.py ```& ```cfsr_dwn_sfc_modified.py```
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

Then edit the date lines in the automation script: 
```
bash_auto_run
```
