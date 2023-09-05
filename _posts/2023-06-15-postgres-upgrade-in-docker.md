---
title:  "Upgrading Postgres when running in Docker"
date:   2023-06-15
category: software
tags: [postgreSQL, docker, upgrade]
---
Major releases of Postgres require upgrading with migration of system tables to new formats. There are multiple ways to upgrade Postgres including pg_upgrade, logical replication, and dump/restore. In this tutorial you will learn how to perform the dump/restore method.

# Preparing for migration

## Password Hashing
Newer hub.docker images use SCRAM-SHA-256 by default in _pg_hba.conf_, this configuration will prevent Roles using md5 hashing from authenticating.

If upgrading from Postgres <= 10 or if the environment is still using the old MD5 passport hashing, you should migrate to using SCRAM-SHA-256. Alternatively set your configuration back to _md5_ however this is not recommended and has security implications.

To get a list of Roles with md5 hashed passwords that need to be reset you can execute the following SQL query:

```sql
SELECT rolname FROM pg_authid 
               WHERE rolpassword LIKE 'md5%'
                 AND rolcanlogin;
```

or

```sql
SELECT rolname FROM pg_authid
               WHERE rolpassword NOT LIKE 'SCRAM-SHA-256%'
                 AND rolcanlogin;
```

Passwords can then be reset with **_psql_** using

```postgresql
\password username
```

for each Role.

There are many online tutorials for this process: https://www.cybertec-postgresql.com/en/from-md5-to-scram-sha-256-in-postgresql/

## Volume mapping
The dump/restore method requires that additional disk space equal to the current database size plus a little bit more. A new Docker volume or local disk path should be mapped to the new container. **Do not** map the old data directory to the new container.

# Migration
For all commands below, _${old_pg_container}_ will refer to our source container we wish to upgrade, _${new_pg_container}_ will refer to our upgraded container.

1. Stop all applications writing to the current database. Leave the _${old_pg_container}_ container running.
2. Start the _${new_pg_container}_ with the upgraded version with docker run or by creating a new container via docker-composer, mounting a new persistent data volume
3. Install any extensions into the new container, but do not install the schema with `CREATE EXTENSION`
4. Migrate the SCHEMA and data from _${old_pg_container}_ to _${new_pg_container}_
    ```bash
    docker exec -i ${old_pg_container} su -c "pg_dumpall --clean" postgres \
     | docker exec -i ${new_pg_container} su -c "psql" postgres
    ```
5. Perform a Vacuum to ensure statistics are gathered
    ```bash
    docker exec -i ${new_pg_container} su -c "vacuumdb -a -z" postgres
    ```
6. Stop _${old_pg_container}_
7. (Optional) Rename _${new_pg_container}_ to _${old_pg_container}_ if you do not wish to modify application configuration
    ```bash
    docker rename ${new_pg_container} ${old_pg_container}
    ``` 
    or if running via docker-compose, update the configuration of the old container to point to the new docker image and new data volume.

You should now be able ot start your application(s).