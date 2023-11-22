# Keycloak DB Create and Import pg_dump file to a new database

If you are able to get a good copy of your PG_DATA directory (usually at something like `/var/lib/postgresql/data/`) but need to "move" the data contents (the objects themselves plus their contents i.e. tables with rows of data) then here are some instructions and example containers you can use in order to do that.

These instructions are based on the idea that you will use the `docker-compose.yaml` file in this repository with Docker Compose in order to create your own local database using the old file system, export the contents of the database to a plain text SQL file using the `pg_dump` command, and then you can run that plain text SQL file in your target database.

This example setup uses the database named "root" and the owner/user named "keycloak".

These steps should be followed:

1. Get your hands on the files themselves!

2. Unpack them to the local `./backup/` folder here so that what sits directly under `./backup/` should be the actual database file contents themselves (config files like `postgresql.conf`, etc plus folders like `base`, `global`, etc).

3. Uncomment the `dumpdb` container from the `./docker-compose.yaml` file, adjust as necessary (for example it is currently written to use `postgres:10-alpine` but this should match the version of the original database) and then start the database like this:

```sh
docker-compose up -d
```

4. Open a bash prompt into `dumpdb` like this:

```sh
docker exec -it dumpdb bash -l
```

5. Ensure you can connect to the database and see all of the right data. For example like this:

```sh
# Log in locally with user as specified in POSTGRES_USER
psql -U keycloak

# List all databases and see that the right databases are there
\l

# Connect to and list all tables in a specific database if you like
\connect root
\d

# Optionally read data from some table
select * from public.realm;

# Quit psql
\q
```

6. Export the database of your choice to a file using `pg_dump`:

```sh
# Connect with user "keycloak" and dump "root" database to file /var/lib/postgresql/data/pgdump.sql
pg_dump -U keycloak -d root -f /var/lib/postgresql/data/pgdump.sql
```

7. From there you can exit the Docker container terminal (`exit`) and then copy the dumped file to your local environment.

```sh
docker cp dumpdb:/var/lib/postgresql/data/pgdump.sql ./restore/pgdump.sql
```

8. Tweak the contents of `./restore/pgdump.sql` as necessary. For example, at the top of the file will likely be some server or database-level stuff that you might not want in the new target database (`SET` various parameters, creating database-level extensions, etc) so these can be commented out as needed.

9. Make sure that the right user is being set in the `ALTER ... OWNER TO ...;` commands and/or adjust your target database to match this user.

10. Copy or mount in the `pgdump.sql` file to your target database. In this example `docker-compose.yaml` file you can comment out the `volumes` section of the `keycloakdb` pod and then restart it.

11. Open a prompt in your new target database and run the file into your new database.

```sh
# Check that your file exists and looks good
head -n 30 /tmp/restore/pgdump.sql

# Check that you can connect to your new database with the right POSTGRES_USER
psql -U keycloak

# List all databases and see that the right target database is there and ideally check that the owner matches what you expect
\l
\d

# Quit psql
\q

# Now import the pgdump.sql file into your root database
psql -U keycloak -d root -f /tmp/restore/pgdump.sql

# There should have been a lot of (hopefully) successful commands (ALTER TABLE etc etc)

# Connect to the target database again and see if it looks like your data looks right
psql -U keycloak
\l
\connect root
\d
select * from public.realm;
\q
```

> **Note:** In case of issues you might have to occasionally fix the owner on mounted files like `sudo chown -R $USER backup` or `sudo chown -R $USER conf` or `sudo chown -R $USER restore` in case you get file permission errors when starting the containers.

Ideally in the end you could actually log into Keycloak with this specific solution at http://localhost:8080/auth and see that all of your "restored" data looks right!
