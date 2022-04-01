# SAEON/postgis
The base postgis/postgis image does not have PostGIS-related CLIs enabled. To use PostGIS CLIs that are NOT enabled by default (for example `raster2pgsql`) it's necessary to extend the base image.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents** 

- [Deployment](#deployment)
- [Managing Postgres](#managing-postgres)
  - [User management](#user-management)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Deployment
This repository builds an image and deploys it to postgis.saeon.ac.za

# Managing Postgres

## User management
```sql
-- Create a user
create user "new_user" with encrypted password 'strong_password';

-- Give that user admin access to a particular database, and set default privileges for future objects added
grant all privileges on database some_database to new_user;
alter default privileges in schema public grant all privileges on tables to new_user;
alter default privileges in schema public grant all privileges on sequences to new_user;

-- Adjust privileges for existing objects in public schema
grant all privileges on all tables in schema public to new_user;
grant all privileges on all sequences in schema public to new_user;
```
