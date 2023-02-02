# SAEON/postgis

The base postgis/postgis image does not have PostGIS-related CLIs enabled. To use PostGIS CLIs that are NOT enabled by default (for example `raster2pgsql`) it's necessary to extend the base image.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Local development](#local-development)
- [Deployment](#deployment)
- [Managing PostgreSQL](#managing-postgresql)
  - [psql](#psql)
    - [Interactively](#interactively)
    - [Non-interactively](#non-interactively)
  - [User management](#user-management)
  - [Backups](#backups)
    - [Take a backup](#take-a-backup)
    - [Restore a backup](#restore-a-backup)
- [Useful operations](#useful-operations)
  - [Loading shapfiles (GDAL ogr2ogr)](#loading-shapfiles-gdal-ogr2ogr)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Local development

The easiest way to work with PostGIS locally is to run the server as a Docker container

```sh
# Create a Docker network
docker network create --driver bridge saeon_local

# Create a PostGIS container
docker run \
  --name postgis \
  --net=saeon_local \
  --restart always \
  -v /home/$USER:/home/$USER \
  -p 5432:5432 \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -d \
  ghcr.io/saeon/postgis:latest

# Run PGAdmin4 (I recommend DBeaver instead actually)
# Connection details
#### host: postgis
#### DB:   postgres
#### User: admin
#### Pswd: password
docker run \
  --rm \
  --net=saeon_local \
  --name pgadmin \
  -p 5001:80 \
  -e PGADMIN_DEFAULT_EMAIL=<your email address> \
  -e PGADMIN_DEFAULT_PASSWORD=password \
  -d \
  dpage/pgadmin4:latest
```

# Deployment
The official PostGIS image doesn't include PostGIS binaries such as `raster2pgsql`. To use PostGIS binaries via CLIs within Docker containers, the official Docker image is extended slightly (refer to [src/Dockerfile](/src/Dockerfile)), and when built hosted as a package in this repository ([ghcr.io/saeon/postgis:latest](https://github.com/SAEON/postgis/pkgs/container/postgis)).

The SAEON PostGIS image is deployed as a Docker swarm service via GitHub Actions (refer to the [stack configuration file](/src/stack.yml)).

# Managing PostgreSQL

## psql

### Interactively
You can start the PSQL cli for interactive use either by logging into an active container (`docker exec -it ...`), or running the CLI via a Docker container. To do this:

```sh
# Assuming a Docker network called `pg`
docker \
  run \
  -it \
  --rm \
  --net=pg \
  ghcr.io/saeon/postgis:latest \
    psql postgres://user:password@host:port/db
```

### Non-interactively
It's also useful to be able to start long-running jobs non-interactively. For example, building an index that may take a long time:

```sh
# Assuming the PostGIS container is on an attachable network called 'pg'
# The host is the name of a Docker stack service
# Omit --rm if you would like to view the psql logs of the completed container afterwards
docker \
  run \
  -d \
  -v /home/$USER:/home/$USER \
  --rm \
  --net pg \
  ghcr.io/saeon/postgis:latest \
    sh -c \
      "psql \
        postgresql://user:password@host:port/db \
        -c 'create index if not exists ....;' \
        -c 'some other sql cmd;'"
```

## User management

**_Create a new admin user for a database_**

```sql
-- Log into PostgreSQL via an admin user to the postgres database
set search_path to postgres;
select current_database(); -- Should show postgres

-- Create a user with all privileges on a new database
create user "username" with encrypted password 'strongpassword';
grant all privileges on database "db_name" to "username";

-- Give that user admin access to a particular database
set search_path to db_name;
select current_database(); -- Should show your new db
grant all privileges on schema public to "username";
grant all privileges on all tables in schema public to "username";
grant all privileges on all sequences in schema public to "username";
alter default privileges in schema public grant all on tables to "username";
alter default privileges in schema public grant all on sequences to "username";
alter default privileges in schema public grant all on functions to "username";
```

**_Revoke a user from a particular database_**

```sql
-- Log into PostgreSQL via an admin user to database you want to remove the user from
set search_path to db_name;
select current_database(); -- Should show the DB you wan't to remove the user from

-- Revoke access to all DB objects
revoke all privileges on all sequences in schema public from "username";
revoke all privileges on all tables in schema public from "username";
revoke all privileges on schema public from "username";
alter default privileges for role "username" in schema public revoke all on tables from "username";
alter default privileges for role "username" in schema public revoke all on sequences from "username";
alter default privileges for role "username" in schema public revoke all on functions from "username";

-- Revoke access to the database
set search_path to postgres;
select current_database(); -- Should show postgres
revoke all privileges on database "db_name" from "username";
```

## Backups
Use the [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html) client to take a backup, and the [pg_restore](https://www.postgresql.org/docs/current/app-pgrestore.html) client to restore a backup. In the context of SAEON's deployment the easiest way to run these clients is via a PostgreSQL Docker container. The instructions below show how to achieve this. Basically the approach is to mount a directory from the host into a Docker container attached to the same network as another Docker container running the target PostgreSQL instance. Run the backup/restore clients against the target PostgreSQL instance and read/write to the mounted directory.

Alternatively, you could install `pg_dump` or `pg_restore` on the host and then use these clients directly. But it is easier to write documentation for the Dockerized approach (such as below) and also easier to ensure that the correct client versions are used compared to the PostgreSQL server version.

Note the following:

- The hostname is the name of the service as defined in the [stack compose file](/src/stack.yml)
- The port is the published port of the container (not the internal port)

### Take a backup
Assuming the target PostgreSQL Docker container is available on a network called `pg`:

```sh
cd ~

# Omit --rm if you would like to view the pg_dump logs of the finished container afterwards
docker \
  run \
  -d \
  -v /home/$USER:/home/$USER \
  --rm \
  --net pg \
  ghcr.io/saeon/postgis:latest \
    sh -c \
      "pg_dump \
        postgresql://user:password@host:port/db \
        --format custom \
        -Z 9 \
          > /home/$USER/db_bak"
```

### Restore a backup
Assuming a backup was taken with the above command, and the target PostgreSQL Docker container is available on a network called `pg`:

```sh
cd ~

# Omit --rm if you would like to view the pg_dump logs of the finished container afterwards
docker \
  run \
  -d \
  -v /home/$USER:/home/$USER \
  --rm \
  --net pg \
  ghcr.io/saeon/postgis:latest \
    sh -c \
      "pg_restore \
        -d postgresql://user:password@host:port/db \
        -j 12 \
        /home/$USER/db_bak"
```

# Useful operations
Some useful commands for interacting with PostGIS instances

## Loading shapfiles (GDAL ogr2ogr)
```sh
docker run \
  -v /home/$USER/:/home/$USER/ \
  --rm \
   osgeo/gdal:ubuntu-full-3.5.1 \
    ogr2ogr \
    --config PG_USE_COPY YES \
    -skipfailures \
    -overwrite \
    -gt 100000 \
    -f PostgreSQL \
    -lco LAUNDER=NO \
    -lco PRECISION=NO \
    -nlt PROMOTE_TO_MULTI \
    -nln mpas \
    "PG:host=hostname port=5432 user=username password=password dbname=dbname active_schema=public" \
    /vsizip/home/$USER/path/to/shapefile.zip
```