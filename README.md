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
# Create a Docker network
docker network create --driver bridge postgis

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
  --rm \
  --net=postgis \
  --name pgadmin \
  -p 5001:80 \
  -e PGADMIN_DEFAULT_EMAIL=<your email address> \
  -e PGADMIN_DEFAULT_PASSWORD=password \
  -d \
  dpage/pgadmin4:latest
```

# Deployment
This repository builds an image and deploys it to postgis.saeon.ac.za

# Managing Postgres

## psql
You can run the `psql` command via Docker

**_Access a local PostgrSQL server_**
```sh
docker \
  run \
  -it \
  --rm \
  --net=postgis \
  ghcr.io/saeon/postgis:14 \
    psql postgres://username:pswd@<hostname>:5432/postgres
```

**_Access a remote PostgrSQL server_**
```sh
docker \
  run \
  -it \
  --rm \
  ghcr.io/saeon/postgis:14 \
    psql postgres://admin:password@<hostname>:5432/postgres
```

## User management
```sql
-- Create a user
create user "username" with encrypted password 'strongpassword';

-- Give that user admin access to a particular database
grant all privileges on database "db_name" to "username";
grant all privileges on all tables in schema public to "username";
grant all privileges on all sequences in schema public to "username";
alter default privileges in schema public grant all on tables to "username";
alter default privileges in schema public grant all on sequences to "username";
alter default privileges in schema public grant all on functions to "username";

-- Revoke access
revoke all privileges on database "db_name" from "username";
revoke all privileges on all tables in schema public from "username";
revoke all privileges on all sequences in schema public from "username";
alter default privileges for role "username" in schema public revoke all on tables from "username";
alter default privileges for role "username" in schema public revoke all on sequences from "username";
alter default privileges for role "username" in schema public revoke all on functions from "username";
```
