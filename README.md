# GIS workshop

## Basic R spatial workflow

### **Spatial set-up**

```{r message=FALSE}

# load packages
library(sf) # vector data
library(terra) # raster data
library(tmap) # visualization

# read in occurrence data
RS <- read.csv("RS_data.csv")

# turn dataframe into spatial object in WGS84
RS_pts <- st_as_sf(RS, coords = c("decimalLongitude", "decimalLatitude"), crs = 4326)

```

Important: You need to know existing CRS here when turning the dataframe into a spatial object--this is what projection the data is already in, not what projection you want it to be in. Also, note that longitude is x, and latitutde is y. The CRS in this command can be set equal to an EPSG code (4326) or a PROJ string, which looks something like: `+proj=utm +zone=11 +datum=WGS84 +units=m +no_defs +ellps=WGS84 +towgs84=0,0,0`

More info here: <https://inbo.github.io/tutorials/tutorials/spatial_crs_coding/>

### **Extract raster values to points**

```{r}

# project spatial points into new CRS (NAD83)
RS_proj <- st_transform(RS_pts, crs = st_crs(4269)) # this is another EPSG code

# read in rasters of future climate variables for 2071-2100, SSP2-45
raster_files <- list.files(path = "climate_data", 
								full.names = TRUE)
raster_stack <- rast(raster_files)

# project raster_stack to match the CRS of the points
raster_stack_proj <- project(raster_stack, crs(RS_proj))

# extract raster values to points
values <- extract(raster_stack_proj, RS_proj)

```

This will output a dataframe with the `RS_proj` coordinate data and columns for each climate variable in `raster_stack` with the values at those coordinates.

### **Make a map**

```{r}

# read in shapefile for map
n_america <- read_sf("north_america_shp/bound_p.shp")

# project polygons into new CRS (NAD83)
n_america_proj <- st_transform(n_america, crs = st_crs(4269))

# subset to relevant locations (you can pretty much just treat this like a dataframe)
new_england <- n_america_proj[which(n_america_proj$STATEABB %in% c("US-ME", "US-NH", "US-VT", "US-MA", "US-RI", "US-CT")),]

# clip raster stack to extent of polygons
raster_stack_crop <- crop(raster_stack_proj, new_england) # crop to extent in box
raster_stack_clip <- terra::mask(raster_stack_crop, vect(new_england)) # crop to exact boundaries of polygons

# clip points to extent of polygons
RS_clip <- st_intersection(RS_proj, new_england)

# make a map
tm_shape(new_england) + 
	tm_polygons(col = "lightgrey", border.col = "white") +
  tm_shape(RS_clip) +
  tm_dots(size = 0.1, col = "darkgreen") +
  tm_layout(legend.position = c("left", "top"), frame = FALSE)

```

"Something has gone wrong! I hate R spatial! What’s happening!?"

-   **Things are not in the same coordinate reference system.** You might have read in your data in the wrong CRS. Or one data source might be in a lat/lon CRS, and the other might be in a projected CRS with units in meters.

-   **Those points are in the ocean.** If you extracted raster values for a bunch of points and some came back with NA values, those points may be outside the raster extent or actually located in a body of water (i.e. no raster value for soil density). If you are so sure that tree is actually on an island in the middle of the lake, you either have a CRS issue causing the tree point to be slightly offset to its actual location relative to the raster, or your raster just doesn’t have that kind of resolution to have values for a tiny island, or your point doesn’t have sufficient coordinate precision (centroid of a 1 km pixel).

-   **You’ve switched latitude and longitude.** Don’t forget that longitude is x, and latitude is y. Or one of them is missing a negative sign because the original data source specified longitude as 72 W instead of -72. Or latitude and longitude were written in degrees minutes seconds notation instead of decimal notation in the original data source.

-   **You’re having package conflicts.** For example, the new package `terra` has a `mask` function that conflicts with the old package `raster`, which has a function with the same name. In this case, detach the old package or specify which package you want: `terra::mask()`.

-   **The visualization package you’ve chosen does not do that.** Have you spent two hours trying to change the font on your legend title in `tmap`? Sadly, `tmap` may just not have that graphic functionality, and you may need to try one of the many other visualization packages: `ggmap`, `leaflet`, `mapview`, etc.

-   **Something outside of R is broken or disconnected.** You can’t load in an OpenStreetMap basemap because Wright’s RStudio isn’t playing nice with Java today. Or somebody needs to update a package on the VACC. Or Google Earth Engine’s API is down. This is the hardest type of error to understand and address, and I typically either try again the next day and it’s fine, or I give up and figure out a different way to do that thing without using the external broken piece.

### Resources

-   [Geocomputation with R](https://r.geocompx.org/adv-map)

-   [Spatial Statistics for Data Science: Theory and Practice with R](https://www.paulamoraga.com/book-spatial/index.html)

-   [Intro to GIS and Spatial Analysis](https://mgimond.github.io/Spatial/index.html) (both ArcGIS Pro and R)

-   [CRAN Task View: Analysis of Spatial Data](https://cran.r-project.org/web/views/Spatial.html)

-   [Milos Popovic’s blog on data viz in R](https://milospopovic.net/blog)
