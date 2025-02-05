Preproccessing access layers v3
================

## About

This script is the 3rd iteration (although v2 didn’t really progress
very far) of a script to use OSM data and downloaded offline data to
‘pre-process’ the access information we have (footpaths, greenspaces
etc.) into a single raster file at 100x100m resolution which indicates
whether that 100x100m square is accesible or not.

This script can be run in a few ways:

-   Run interactively in datalabs by running chunks
-   Using rmarkdown `purl` function to export chunks to `.R` scripts
    which can be run as jobs in Rstudio
    (\`/preproc\_jobs/preprocessing\_access\_laters\_job\[1-8\].R) -
    this is actually rather slow so not really using this approach
-   Use JASMIN/SLURM to schedule using the `slurm_pre_proc.sbatch` file

These scripts are generated from this main script using the code in the
Purling subheading below.

There is also the script `download_osm_data` which is a copy of the
chunk `data_prep` from `preprocessing_access_layers_v3.Rmd` to run from
the command line on JASMIN to download the OSM files for pre-processing
access layers

## Purling

Here we’ve got two chunks that generate the `.R` scripts for Rstudio
jobs and JASMIN/SLURM.

You can see that it purls the script but then alters the first line of
the scrpit to assign a job ID or or for the JASMIN script it ensure that
it gets arguments from the command line call. We can then use the
`exists()` function to check if the script is being run in datalabs or
JASMIN to do different behaviors like where to set as a working
directory.

Convert R markdown chunks into R script for running local jobs

``` r
setwd("/data/notebooks/rstudio-conlayersimon/DECIDE_constraintlayers")
for (i in 1:8){
  knitr::purl("Scripts/preprocessing_access_layers_v3.Rmd",output = paste0( "Scripts/preproc_jobs/preprocessing_access_layers_job",i,".R"))
  
  lines <- readLines(paste0("Scripts/preproc_jobs/preprocessing_access_layers_job",i,".R"))
  lines[1] <- paste("job_id <-",i)
  
  write(lines,file=paste0("Scripts/preproc_jobs/preprocessing_access_layers_job",i,".R"))
}
```

Convert R markdown chunks into R script that runs for one file (for
JASMIN)

``` r
setwd("/data/notebooks/rstudio-conlayersimon/DECIDE_constraintlayers")
knitr::purl("Scripts/preprocessing_access_layers_v3.Rmd",output = "Scripts/preproc_jobs/preprocessing_access_layers_job_slurm.R")

lines <- readLines("Scripts/preproc_jobs/preprocessing_access_layers_job_slurm.R")
lines[1] <- 'args = commandArgs(trailingOnly=TRUE); slurm_grid_id <- args[1]'

write(lines,file="Scripts/preproc_jobs/preprocessing_access_layers_job_slurm.R")
```

Finally, here starts the actual script…

## Set up

``` r
#THIS LINE IS REPLACED WITH JOB ID ASSIGNMENT if generating job files
#load packages
library(osmextract) #for using overpass API
library(sf) #for all the geographic operations
library(leaflet) # for making maps for sanity checking
library(raster)
library(rgdal)
library(dplyr)
library(htmlwidgets)
```

Set working directories and file directories for datalabs or JASMIN

``` r
if(exists("job_id")){
  #datalabs
  setwd(file.path("","data","notebooks","rstudio-conlayersimon","DECIDE_constraintlayers","Scripts"))
  
  raw_data_location <- file.path("","data","data","DECIDE_constraintlayers","raw_data")
  processed_data_location <- file.path("","data","data","DECIDE_constraintlayers","processed_data")
  environmental_data_location <- file.path("","data","data","DECIDE_constraintlayers","environmental_data")
}


if(exists("slurm_grid_id")){
  #JASMIN
  setwd(file.path("","home","users","simrol","DECIDE","DECIDE_constraintlayers","Scripts"))
  
  raw_data_location <- file.path("","home","users","simrol","DECIDE","raw_data")
  processed_data_location <- file.path("","home","users","simrol","DECIDE","processed_data")
  environmental_data_location <- file.path("","home","users","simrol","DECIDE","environmental_data")
}
```

## Data preparation

### Downloading and processing OSM data into GeoPackage format

This onnly needs to be run once, and you’ll see isn’t purled into the
scripts used by JASMIN

``` r
#set the extent of the area to get data from (use a geofabrik region name)
download_area <- "Great Britain"
#download_area <- "North Yorkshire" # use a smaller area for testing

#Access lines
vt_opts_1 = c(
    "-select", "osm_id, highway, designation, footway, sidewalk",
    "-where", "highway IN ('footway', 'path', 'residential','unclassified','tertiary','sidewalk') OR designation IN ('public_footpath','byway_open_to_all_traffic','restricted_byway','public_bridleway','access_land') OR footway = 'sidewalk' OR sidewalk IN ('both','left','right')"
  )

oe_get(
  download_area,
  layer = "lines",
  provider = "geofabrik",
  match_by = "name",
  max_string_dist = 1,
  level = NULL,
  download_directory = file.path(raw_data_location,"OSM"),
  force_download = F,
  vectortranslate_options = vt_opts_1,
  extra_tags = c("designation","footway","sidewalk"),
  force_vectortranslate = T,
  quiet = FALSE
) %>% st_write(file.path(raw_data_location,"OSM","access_lines.gpkg"),delete_layer = T)


# No-go lines
vt_opts_2 <- c(
    "-select", "osm_id, highway, railway",
    "-where", "highway IN ('motorway','trunk') OR railway = 'rail'"
  )

oe_get(
  download_area,
  layer = "lines",
  provider = "geofabrik",
  match_by = "name",
  max_string_dist = 1,
  level = NULL,
  download_directory = file.path(raw_data_location,"OSM"),
  force_download = F,
  vectortranslate_options = vt_opts_2,
  extra_tags = c("railway"),
  force_vectortranslate = T,
  quiet = FALSE
) %>% st_write(file.path(raw_data_location,"OSM","no_go_lines.gpkg"),delete_layer = T)


#no go areas
vt_opts_3 <- c(
    "-select", "osm_id, landuse, aeroway",
    "-where", "landuse IN ('quarry','landfill','industrial','military') OR aeroway = 'aerodrome'"
  )

oe_get(
  download_area,
  layer = "multipolygons",
  provider = "geofabrik",
  max_string_dist = 1,
  level = NULL,
  download_directory = file.path(raw_data_location,"OSM"),
  force_download = F,
  vectortranslate_options = vt_opts_3,
  force_vectortranslate = T,
  quiet = FALSE
) %>% st_write(file.path(raw_data_location,"OSM","no_go_areas.gpkg"),delete_layer = T)


# water areas
vt_opts_4 <- c(
    "-select", "osm_id, natural",
    "-where", "natural = 'water'"
  )

oe_get(
  download_area,
  layer = "multipolygons",
  provider = "geofabrik",
  max_string_dist = 1,
  level = NULL,
  download_directory = file.path(raw_data_location,"OSM"),
  force_download = F,
  vectortranslate_options = vt_opts_4,
  force_vectortranslate = T,
  quiet = FALSE
) %>% st_write(file.path(raw_data_location,"OSM","water_areas.gpkg"),delete_layer = T)
```

Demonstration of accessing the processed OSM data. This just shows how
the OSM data is loaded in by using `st_read` and using a well known text
filter to only extract the data for this area (rather than generating
sepearate 10km grid squares - which would load faster but just measn
you’ve got more files floating about).

I use leaflet to generate a map to ensure that it’s looking sensible and
can try out different grid squares - it’s a good sanity check that the
data is in the right projection, and that the OSM features we’ve
downloaded are the sorts of things we want use as access information.

``` r
uk_grid <- st_read(file.path(raw_data_location,"UK_grids","uk_grid_10km.shp"))
st_crs(uk_grid) <- 27700
this_10k_grid <- uk_grid[1313,]$geometry #get the 10kgrid
this_10k_gridWGS84 <- st_transform(this_10k_grid, 4326) %>% st_as_text()# convert to WGS84

layer1 <- st_read(file.path(raw_data_location,"OSM","access_lines.gpkg"),
                        wkt_filter = this_10k_gridWGS84
                        )

layer2 <- st_read(file.path(raw_data_location,"OSM","no_go_lines.gpkg"),
                        wkt_filter = this_10k_gridWGS84
                        )

layer3 <- st_read(file.path(raw_data_location,"OSM","no_go_areas.gpkg"),
                        wkt_filter = this_10k_gridWGS84
                        )


layer4 <- st_read(file.path(raw_data_location,"OSM","water_areas.gpkg"),
                        wkt_filter = this_10k_gridWGS84
                        )

m <- leaflet() %>%
  addTiles() %>%
  addPolylines(data = layer1,weight = 1) %>%
  addPolylines(data = layer2,weight = 1,color = "red") %>%
  addPolygons(data = layer3,weight = 1,fillColor ="red") %>%
  addPolygons(data = layer4,weight = 1) %>%
  addPolygons(data=this_10k_grid %>% st_transform(4326) ,opacity=1,fillOpacity = 0,weight=2,color = "black")
  
m
```

### Accessing offline data sources

This block defines a big function for getting offline files for a 10km
grid number. In the `file_locations` vector you can define the file
paths to look for shapefiles. If we add more data they need to be added
here. Those shapefiles needs to have a number in their filename that
indicates which grid square (1-3025) the data is for.

``` r
#base location
base_location <- file.path(raw_data_location,"")

#folders
file_locations <- c(
  'CRoW_Act_2000_-_Access_Layer_(England)-shp/gridded_data_10km/',
  'OS_greenspaces/OS Open Greenspace (ESRI Shape File) GB/data/gridded_greenspace_data_10km/',
  #'SSSIs/gridded_data_10km/',
  'greater-london-latest-free/london_gridded_data_10km/',
  'rowmaps_footpathbridleway/rowmaps_footpathbridleway/gridded_data_10km/',
  #'RSPB_Reserve_Boundaries/gridded_data_10km/',
  'national_trust/gridded_data_10km/'
)

# function to get the data
get_offline_gridded_data <- function(grid_number){
  returned_files <- list()
  
  for (ii in 1:length(file_locations)){
    file_location <- file_locations[ii]
    all_grids <- list.files(file.path(base_location,file_location))
    
    right_file <- all_grids[grep(paste0("_",grid_number,".shp"), all_grids)]
    
    if (length(right_file)>0){
      #build file path
      file_path <- paste0(base_location,file_location,right_file)
      
      #load in file
      file_shp <- st_read(file_path, quiet = TRUE) %>% st_transform(4326)
      
      #add it to the object that this function returns
      returned_files[[length(returned_files)+1]] <- file_shp
    }
  }
  #return the files
  returned_files
}

# test <- get_offline_gridded_data(1313)
# 
# #check access with lapply
check_access_lapply <- function(...){
  st_is_within_distance(...) %>% rowSums()
}
# requires taster_as_sf object which is defined later in the script so will error if you run this the script in order
# test2 <- lapply(test,check_access_lapply,x = raster_as_sf,dist = 100,sparse = FALSE) %>% Reduce(f='+')
```

## Data processing

This is where the main processing happens, taking all the access layers
and the coordinates (from a raster) for the centroid of each 100mx100m
grid square and determining if each centroid is near a good accesble
feature - or a bad feature like water or land fill site.

The function takes 4 arguments - `grid_number` - the number of the grid
from 1 to 3025 - `grids` - the UK 10km grids - `raster_df` - a data
frame of the 100x100m grid square centroids in easting and northings -
`produce_map` - if `TRUE` the function will return a leaflet map instead
of the

Setting this up as a function means to be able to be run as jobs.

``` r
sf::sf_use_s2(T)

#load in required to do the job

uk_grid <- st_read(file.path(raw_data_location,'UK_grids','uk_grid_10km.shp'))
st_crs(uk_grid) <- 27700

# the big raster
raster100 <- raster::stack(file.path(environmental_data_location,'100mRastOneLayer.grd'))
#plot(raster100)
#get the CRS for when we project back to raster from data ramew
raster_crs <- st_crs(raster100)
raster100_df <- as.data.frame(raster100, xy=T, centroids=TRUE)[,1:2]


#for testing:
#grid_number <- 1516
#grids <- uk_grid
#raster_df <- raster100_df

#define the function for assessing accessibility
assess_accessibility <- function(grid_number,grids,raster_df,produce_map = F){
  
  #load 10k grid
  this_10k_grid <- grids[grid_number,]$geometry #get the 10kgrid
  this_10k_gridWGS84 <- st_transform(this_10k_grid, 4326) # convert to WGS84
  grid_bb <- st_bbox(this_10k_grid)
  
  #get the centroids of the the 100m grids within the 10k grid
  raster_this_grid <- raster_df %>% filter(x > grid_bb$xmin,
                                              x < grid_bb$xmax,
                                              y > grid_bb$ymin,
                                              y < grid_bb$ymax)
  
  #get the easting and northing projection
  projcrs <- st_crs(this_10k_grid)
  
  # make the raster df into a sd object using the projection but transform it to WGS84 for these operations
  raster_as_sf <- st_as_sf(raster_this_grid,coords = c("x", "y"),crs = projcrs) %>% st_transform(4326)
  
  this_10k_gridWGS84_wkt <- this_10k_gridWGS84 %>% st_as_text()
  
  # load OSM data
  osm_layer1 <- st_read(file.path(raw_data_location,"OSM","access_lines.gpkg"),
                        wkt_filter = this_10k_gridWGS84_wkt,
                        quiet = T
                        )

  osm_layer2 <- st_read(file.path(raw_data_location,"OSM","no_go_lines.gpkg"),
                        wkt_filter = this_10k_gridWGS84_wkt,
                        quiet = T
                        )

  osm_layer3_no_go <- st_read(file.path(raw_data_location,"OSM","no_go_areas.gpkg"),
                        wkt_filter = this_10k_gridWGS84_wkt,
                        quiet = T
                        ) %>% filter(landuse != "military")
  
  osm_layer3_warn <- st_read(file.path(raw_data_location,"OSM","no_go_areas.gpkg"),
                        wkt_filter = this_10k_gridWGS84_wkt,
                        quiet = T
                        ) %>% filter(landuse == "military")

  osm_layer4 <- st_read(file.path(raw_data_location,"OSM","water_areas.gpkg"),
                        wkt_filter = this_10k_gridWGS84_wkt,
                        quiet = T
                        )
  
  
  # Load offline data
  offline_good_features_data <- get_offline_gridded_data(grid_number)
  
  
  #vectors for filling with info
  distance_check_bad <- distance_check_good <- distance_check_warning <- distance_check_water <- rep(0,nrow(raster_this_grid))
  
  #checking access for offline layers
  #first try using lapply because it's quickest but then there's an alternative approch which is slower by copes with the spherical geometry issue that sometimes occurs
  tryCatch({
    if (length(offline_good_features_data)>0){
      distance_check_good <- lapply(offline_good_features_data,check_access_lapply,x = raster_as_sf,dist = 100,sparse = FALSE) %>% Reduce(f='+')
    }
  }, error = function(err) {
    sf::sf_use_s2(F)
    
    
    if (length(offline_good_features_data)>0){
      for (ii in 1:length(offline_good_features_data)){
        distance_check_good <- distance_check_good + rowSums(st_is_within_distance(raster_as_sf,offline_good_features_data[[ii]]$geometry,dist = 100,sparse = FALSE))
      }
    }

    sf::sf_use_s2(T)
  })
  
  
  tryCatch({
    #check OSM data
    #good lines
    distance_check_good <- distance_check_good + rowSums(st_is_within_distance(raster_as_sf,osm_layer1,dist = 100,sparse = FALSE))  
    
    #bad lines and areas
    distance_check_bad <- distance_check_bad + rowSums(st_is_within_distance(raster_as_sf,osm_layer2,dist = 50,sparse = FALSE)) + rowSums(st_is_within_distance(raster_as_sf,osm_layer3_no_go,dist = 25,sparse = FALSE)) 
    
    # warning areas
    distance_check_warning <- distance_check_warning + rowSums(st_is_within_distance(raster_as_sf,osm_layer3_warn,dist = 25,sparse = FALSE))
    
    #Water
    distance_check_water <- distance_check_water + rowSums(st_is_within_distance(raster_as_sf,osm_layer4,dist = 0,sparse = FALSE))
    
  }, error = function(err){
    #same as before but without spherical geometry
    sf::sf_use_s2(F)
    distance_check_good <- distance_check_good + rowSums(st_is_within_distance(raster_as_sf,osm_layer1,dist = 100,sparse = FALSE))  
    distance_check_bad <- distance_check_bad + rowSums(st_is_within_distance(raster_as_sf,osm_layer2,dist = 50,sparse = FALSE)) + rowSums(st_is_within_distance(raster_as_sf,osm_layer3_no_go,dist = 25,sparse = FALSE)) 
    distance_check_warning <- distance_check_warning + rowSums(st_is_within_distance(raster_as_sf,osm_layer3_warn,dist = 25,sparse = FALSE))
    distance_check_water <- distance_check_water + rowSums(st_is_within_distance(raster_as_sf,osm_layer4,dist = 0,sparse = FALSE))
    sf::sf_use_s2(T)
  })
  
  # water buffering options (currently not used)
  # if(sum(distance_check_water>0)>0){
  #   buffered_water_points <- raster_as_sf[distance_check_water>0,] %>% st_buffer(dist=50)
  #   points_in_water <- st_covered_by(buffered_water_points,osm_layer4,sparse = F) %>% rowSums()
  #   rownames(buffered_water_points)[points_in_water>0]
  #   distance_check_water <- 0
  #   distance_check_water[rownames(raster_as_sf) %in% rownames(buffered_water_points)[points_in_water>0]] <- 1
  # }
  
  raster_as_sf$access <- distance_check_good
  raster_as_sf$no_go <- distance_check_bad
  raster_as_sf$warning <- distance_check_warning
  raster_as_sf$water <- distance_check_water
  
  raster_as_sf$composite <- 0.5 # neutral
  raster_as_sf$composite[raster_as_sf$no_go>0 | raster_as_sf$warning > 0] <- 0 #no go
  raster_as_sf$composite[raster_as_sf$access>0 & raster_as_sf$no_go==0] <- 1 #go
  raster_as_sf$composite[raster_as_sf$access>0 & raster_as_sf$warning > 0] <- 0.75 # warning
  raster_as_sf$composite[raster_as_sf$water>0] <- 0.25 #water
  
  if(produce_map){
    m <- leaflet() %>%
      addTiles() %>%
      addPolylines(data = osm_layer1,weight = 1) %>%
      addPolylines(data = osm_layer2,weight = 1,color = "red") %>%
      addPolygons(data = osm_layer3_warn,weight = 1,fillColor ="orange") %>%
      addPolygons(data = osm_layer3_no_go,weight = 1,fillColor ="red") %>%
      addPolygons(data = osm_layer4,weight = 1) %>%
      addPolygons(data=this_10k_grid %>% st_transform(4326) ,opacity=1,fillOpacity = 0,weight=2,color = "black") %>%
      addCircles(data = raster_as_sf %>% filter(composite==1),radius = 50,weight=0,color = "blue") %>%
      addCircles(data = raster_as_sf %>% filter(composite==0),radius = 50,weight=0,color = "red") %>%
      addCircles(data = raster_as_sf %>% filter(composite==0.75),radius = 50,weight=0,color = "orange") %>%
      addCircles(data = raster_as_sf %>% filter(composite==0.25),radius = 50,weight=0,color = "white")
    
    return(m)
  }
  
  
  raster_df[raster_df$x > grid_bb$xmin & raster_df$x < grid_bb$xmax & raster_df$y > grid_bb$ymin & raster_df$y < grid_bb$ymax,"access"] <- raster_as_sf$composite
  
  raster100_to_save <- raster_df %>% filter(x > grid_bb$xmin,
                                               x < grid_bb$xmax,
                                               y > grid_bb$ymin,
                                               y < grid_bb$ymax)
  
  return(raster100_to_save)
}

#1313 is near Ladybower reservoir in the peak district
#test <- assess_accessibility(1313,uk_grid,raster100_df,produce_map = F)
```

## Saving outputs

### If executed as an Rstudio job

Build a dataframe called `log_df` that records how long it took to
process each grid square

``` r
if(exists("job_id")){
  #set it all off as 8 seperate jobs, saving the individual 10k grids
  log_df <- data.frame(grid_no = 0,time_taken = "",time = "")[-1,]
  job_sequence <- seq.int(from = 1,to = 3025,length.out = 9)
  
  for (i in job_sequence[job_id]:job_sequence[job_id+1]){
    cat(paste('\r grid:',i," progress:",round((i-job_sequence[job_id])/378*100),"%"))
    time_taken <- system.time({
      access_raster <- assess_accessibility(i,uk_grid,raster100_df,produce_map = F)
      })
    
    log_df[nrow(log_df)+1,] <- c(i,time_taken[3],Sys.time()%>% toString())
    
    saveRDS(access_raster,file = paste0("/data/data/DECIDE_constraintlayers/processed_data/access_raster_grid",i,".RDS"))
    
    saveRDS(log_df,file = paste0("/data/data/DECIDE_constraintlayers/processed_data/logs/log_",job_id,".RDS"))
  }
}
```

### If exectuted as a SLURM script

Just save the file and print the time taken

``` r
if(exists("slurm_grid_id")){
  
  time_taken <- system.time({
    access_raster <- assess_accessibility(slurm_grid_id,uk_grid,raster100_df,produce_map = F)
    })
    
  saveRDS(access_raster,file = paste0(processed_data_location,"/access_raster_grid",slurm_grid_id,".RDS"))
  
  print(time_taken)
}
```
