# SAEON/postgis
The base postgis/postgis image does not have PostGIS-related CLIs enabled. To use PostGIS CLIs that are NOT enabled by default (for example `raster2pgsql`) it's necessary to extend the base image.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents** 

- [Local development](#local-development)
- [Deployment](#deployment)
- [Managing Postgres](#managing-postgres)
  - [User management](#user-management)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Local development
The easiest way to work with PostGIS locally is to run the server as a Docker container

```sh
# Create a PostGIS container
docker run \
  --name postgis \
  --net=postgis \
  --restart always \
  -p 5432:5432 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -d \
  ghcr.io/saeon/postgis:latest

# Run PGAdmin4
# Connection details
#### host: postgis
#### DB:   postgres
#### User: admin
#### Pswd: password 
docker run \
  --net=postgis \
  --name pgadmin \
  -p 5001:80 \
  -e PGADMIN_DEFAULT_EMAIL=<your email address> \
  -e PGADMIN_DEFAULT_PASSWORD=password \
  -d \
  dpage/pgadmin4:6.8
```

# Deployment
This repository builds an image and deploys it to postgis.saeon.ac.za

# Managing Postgres

## User management
```sql
-- Create a user
create user "username" with encrypted password 'bcSxs9hhJ3L9HZ';

-- Give that user admin access to a particular database
grant all privileges on database "db_name" to "username";
grant all privileges on all tables in schema public to "username";
grant all privileges on all sequences in schema public to "username";
alter default privileges in schema public grant all on tables to "username"
alter default privileges in schema public grant all on sequences to "username"
alter default privileges in schema public grant all on functions to "username"

-- Revoke access
revoke all privileges on database "db_name" from "username";
revoke all privileges on all tables in schema public from "username";
revoke all privileges on all sequences in schema public from "username";
alter default privileges for role "username" in schema public revoke all on tables from "username";
alter default privileges for role "username" in schema public revoke all on sequences from "username";
alter default privileges for role "username" in schema public revoke all on functions from "username";
```
