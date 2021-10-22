# SAEON/postgis
SAEON PostgreSQL + PostGIS Docker images

The base postgis/postgis image does not have PostGIS-related CLIs enabled. To use PostGIS CLIs that are NOT enabled by default (for example `raster2pgsql`) it's necessary to extend the base image.
