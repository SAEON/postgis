# SAEON/postgis

The base postgis/postgis image does not have PostGIS-related CLIs enabled. To use PostGIS CLIs that are NOT enabled by default (for example `raster2pgsql`) it's necessary to extend the base image.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Local development](#local-development)
- [Deployment](#deployment)
- [Managing PostgreSQL](#managing-postgresql)
  - [psql](#psql)
  - [User management](#user-management)
  - [Create a backup](#create-a-backup)
  - [Restore a backup](#restore-a-backup)

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

This repository builds an image and deploys it to a single node Docker Swarm stack on postgis.saeon.ac.za. The command that is used (useful to include here for testing purposes):

```sh
export $(cat src/stack.env) > /dev/null 2>&1;
docker stack deploy -c src/stack.yml postgis
```

# Managing PostgreSQL

## psql

You can run the `psql` command via Docker

**_Access a local PostgreSQL server_**

```sh
docker \
  run \
  -it \
  --rm \
  --net=saeon_local \
  ghcr.io/saeon/postgis:latest \
    psql postgres://username:pswd@<hostname>:5432/postgres
```

**_Access a remote PostgrSQL server_**

```sh
docker \
  run \
  -it \
  --rm \
  ghcr.io/saeon/postgis:latest \
    psql postgres://admin:password@<hostname>:5432/postgres
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

## Create a backup
Use the [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html) client to take a backup:

```sh
cd ~

# Regular backup
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

# Backup to directory (theoretically this allows for using multiple cores, but I'm not sure how that works with Docker)
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
        --format directory \
        -Z 9 \
        -j 12 \
        -f /home/$USER/db_bak"

# Then to watch progress / view logs
docker container ls
```

## Restore a backup
TODO