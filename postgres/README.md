# PostgreSQL [Master-Slave Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/postgres/images/postgresql-logo.png" width="200" height="200"/>

To know more about postgresql, visit https://www.postgresql.org/

## Overview
From the official docs -

> PostgreSQL is a powerful, open source object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

## Setup
Installation of `PostgreSQL 10` on `Ubuntu 18.04.3 LTS` - [ref](https://www.postgresql.org/download/linux/ubuntu/)

1. Add PostgreSQL repository, update and install postgresql
    ```bash
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
    sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -sc)-pgdg main" > /etc/apt/sources.list.d/PostgreSQL.list'
    sudo apt update
    sudo apt-get install postgresql-10
   
    # Verify installation 
    sudo systemctl stop postgresql.service
    sudo systemctl start postgresql.service
    sudo systemctl enable postgresql.service
    sudo systemctl status postgresql.service
    ```

2. Login and change postgres user password
    ```bash
    sudo su postgres
    psql -U postgres
    ALTER USER postgres WITH PASSWORD 'postgres';
    ```

3. Update `/etc/postgresql/10/main/postgresql.conf` to allow remote connections -
    ```bash
    listen_addresses = '*'
    ```
    
4. Update `/etc/postgresql/10/main/pg_hba.conf` to allow remote authentication -
    ```bash
    # TYPE     DATABASE        USER            ADDRESS METHOD        AUTH_METHOD
    host       all             all             all                   md5
    ```
NOTE: For setting up streaming replication and production grade configuration, refer following [timescale's documentation](../timescale/README.md).

## Misc
1. Install postgis extension - [ref1](https://postgis.net/install/), [ref2](https://trac.osgeo.org/postgis/wiki/UsersWikiPostGIS24UbuntuPGSQL10Apt)
    ```bash
    # List available versions
    sudo apt-cache policy postgis
       # |- sample output
        postgis:
          Installed: (none)
          Candidate: 3.0.1+dfsg-2.pgdg18.04+1
          Version table:
             3.0.1+dfsg-2.pgdg18.04+1 500
                500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
             2.4.3+dfsg-4 500
                500 http://archive.ubuntu.com/ubuntu bionic/universe amd64 Packages
    
    # Install default version
    sudo apt install postgis --no-install-recommends
       # |- sample output
       postgis:
         Installed: 3.0.1+dfsg-2.pgdg18.04+1
         Candidate: 3.0.1+dfsg-2.pgdg18.04+1
         Version table:
        *** 3.0.1+dfsg-2.pgdg18.04+1 500
               500 http://apt.postgresql.org/pub/repos/apt bionic-pgdg/main amd64 Packages
               100 /var/lib/dpkg/status
            2.4.3+dfsg-4 500
               500 http://archive.ubuntu.com/ubuntu bionic/universe amd64 Packages
    
    # If you'd prefer to install a specific version where 2.4.3+dfsg-4 replace with your desired version from cache policy list
    sudo apt install postgis=2.4.3+dfsg-4  --no-install-recommends
    
    # Install scripts 
    sudo apt install postgresql-10-postgis-scripts
    # Otherwise you'll get following error
    # create extension postgis;
    # ERROR:  could not open extension control file "/usr/share/postgresql/10/extension/postgis.control": No such file or directory
    
    # Finally create extension in a db or schema
    CREATE EXTENSION postgis;
    # or 
    CREATE EXTENSION postgis SCHEMA <schema-name>;
 
    e.g.
    test=# CREATE EXTENSION postgis;
    CREATE EXTENSION
    test=# SELECT PostGIS_version();
                postgis_version
    ---------------------------------------
     2.4 USE_GEOS=1 USE_PROJ=1 USE_STATS=1
    (1 row)
    ```

2. Install `wal2json` - a logical decoding plugin - [ref1](https://github.com/eulerto/wal2json), [ref2](https://www.ubuntuupdates.org/package/postgresql/bionic-pgdg/main/base/postgresql-10-wal2json) 
    ```bash
    sudo apt-get install postgresql-10-wal2json
    ```

3. Logical replication slots with `wal2json` as the logical decoding plugin 
    * Configure server to use wal2json along with replication settings in `/etc/postgresql/10/main/postgresql.conf`
        ```bash
        # MODULES
        shared_preload_libraries = 'wal2json'
         
        # REPLICATION
        wal_level = logical             
        max_wal_senders = 1             
        max_replication_slots = 1
        ```       
    * Update `pg_hba.conf` to the following for authenticating `myuser` to use `mydatabase`
        ```bash
        local     mydatabase      myuser      trust
        ```
    * Create/delete slot - [ref](https://www.postgresql.org/docs/10/logicaldecoding-example.html)
        ```bash
        # Create slot
        pg_recvlogical -d <database> --slot <slot-name> --create-slot -P wal2json
     
        # Delete slot
        select pg_drop_replication_slot('<slot-name>');
        ```

## Administration
1. User Management - [[pg-docs-create-user](https://www.postgresql.org/docs/12/sql-createuser.html), [pg-docs-grant](https://www.postgresql.org/docs/12/sql-grant.html), [AWS](https://aws.amazon.com/blogs/database/managing-postgresql-users-and-roles/), [Blog](https://tableplus.com/blog/2018/04/postgresql-how-to-grant-access-to-users.html)]
    ```bash
    Create Role -
    -------------
    CREATE ROLE readonly WITH LOGIN PASSWORD 'test';
    or
    CREATE ROLE readonly;
    ALTER ROLE readonly WITH LOGIN;
    
    Grant Read Role -
    ------------------
    GRANT CONNECT ON DATABASE test_db TO readonly;
    \c test_db;
    GRANT USAGE ON SCHEMA public TO readuser;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO readuser;
    GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO readuser;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readuser;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON SEQUENCES TO readuser;
    
    Drop Read Role -
    ----------------
    REVOKE CONNECT ON Database test FROM readonly;
    \c test_db;
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM readonly;
    REVOKE ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public FROM readonly;
    REVOKE ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public FROM readonly;
    REVOKE ALL PRIVILEGES ON SCHEMA public FROM readonly;
    REASSIGN OWNED BY readonly to postgres;
    DROP OWNED BY readonly; - https://stackoverflow.com/questions/9840955/postgresql-drop-role-fails-because-of-default-privileges
    DROP ROLE readonly;
    
    ========================================================================
    Sample production scenario - 
    i) Create readonly, readwrite roles without login.
    ii) Grant these roles to new users with login.
    iii) PGPASSWORD=$DB_PASSWORD psql -h $DB_HOST -U $DB_ROOT_USER $DATABASE -tAc "select * from rand;"
    ========================================================================
 
    -- Revoke privileges from 'public' role
    REVOKE CREATE ON SCHEMA public FROM PUBLIC;
    REVOKE ALL ON DATABASE mydatabase FROM PUBLIC;
    
    -- Read-only role
    CREATE ROLE readonly;
    GRANT CONNECT ON DATABASE mydatabase TO readonly;
    GRANT USAGE ON SCHEMA myschema TO readonly;
    GRANT SELECT ON ALL TABLES IN SCHEMA myschema TO readonly;
    GRANT SELECT ON ALL SEQUENCES IN SCHEMA myschema TO readonly;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT ON TABLES TO readonly;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT ON SEQUENCES TO readonly;
    
    -- Read/write role
    CREATE ROLE readwrite;
    GRANT CONNECT ON DATABASE mydatabase TO readwrite;
    GRANT USAGE, CREATE ON SCHEMA myschema TO readwrite;
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA myschema TO readwrite;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;
    GRANT USAGE ON ALL SEQUENCES IN SCHEMA myschema TO readwrite;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT USAGE ON SEQUENCES TO readwrite;
    
    -- User creation
    CREATE USER reporting_user1 WITH LOGIN PASSWORD 'some_secret_passwd';
    CREATE USER reporting_user2 WITH LOGIN  PASSWORD 'some_secret_passwd';
    CREATE USER app_user1 WITH LOGIN PASSWORD 'some_secret_passwd';
    CREATE USER app_user2 WITH LOGIN PASSWORD 'some_secret_passwd';
    
    -- User updation
    ALTER USER reporting_user1 WITH PASSWORD 'new_password';
 
    -- Grant privileges to users
    GRANT readonly TO reporting_user1;
    GRANT readonly TO reporting_user2;
    GRANT readwrite TO app_user1;
    GRANT readwrite TO app_user2;
   
   -- Revoke privileges from users
   REVOKE readonly FROM reporting_user1;
   REVOKE readonly FROM reporting_user2;
   REVOKE readwrite FROM app_user1;
   REVOKE readwrite FROM app_user2;

    -- Sample application user
 
    1. Create role and grant permissions 
    CREATE ROLE airflow WITH login PASSWORD 'airflow';
    GRANT CONNECT ON DATABASE airflow TO airflow;
    GRANT ALL PRIVILEGES ON SCHEMA public TO airflow;
    GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO airflow;
    GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO airflow;
    GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO airflow;
    --- The remaning ones are needed to extend permission of user airflow for future objects created by any user
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO airflow;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO airflow;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO airflow;
 
    2. Make this role/user as the owner of the tables (if any), from postgres to airflow
    ALTER TABLE table_name OWNER TO airflow;
    
    3. Revoke previliges
    REVOKE CONNECT ON DATABASE airflow FROM airflow;
    REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM airflow;
    REVOKE ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public FROM airflow;
    REVOKE ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public FROM airflow;
    REVOKE ALL PRIVILEGES ON SCHEMA public FROM airflow;
    REASSIGN OWNED BY airflow to postgres;
    DROP OWNED BY airflow;
    DROP ROLE airflow;
    
    -- Misc
    ALTER ROLE name RENAME TO new_name;
    ALTER ROLE postgres WITH PASSWORD 'postgres';
    ``` 

2. Dump and restore
    ```bash
    # --- 1. Table ---
        # Dump 1 table
        pg_dump -h <hostname> -p 5432 -U <username> -d <db-name> -t <schema>.<table-name> > table.sql
     
        # Restore 1 table
        psql -h <hostname> -p 5432 -U <username> -d <db-name> -n <schema-name> < table.sql
 
    # --- 2. Database ---
        # Dump 1 db
        pg_dump -h <hostname> -p 5432 -U <username> -d <db-name> > db.sql
        
        # Restore 1 db
        psql -h <hostname> -p 5432 -U <username> -d <db-name> < db.sql
        
        # Dump all dbs
        pg_dumpall -h <hostname> -p 5432 -U <username> > dbs.sql
        
        # Restore all dbs
        psql -h <hostname> -p 5432 -U <username> < dbs.sql
    ```

3. Resource size
    ```bash
    # Database
    SELECT pg_size_pretty(pg_database_size('dbname'));
    
    # Tables
    SELECT pg_size_pretty(pg_total_relation_size('tablename'));
    
    # All tables reverse sorted by size
    SELECT
      nspname,
      relname AS "relation",
      pg_size_pretty (pg_total_relation_size (C.oid)) AS "total_size"
    FROM 
      pg_class C
    LEFT JOIN 
      pg_namespace N ON N.oid = C.relnamespace
    WHERE
      nspname NOT IN ('pg_catalog', 'information_schema')
      AND C.relkind != 'i'
      AND nspname !~ '^pg_toast'
    ORDER BY
      pg_total_relation_size (C.oid) DESC
    ```

4. Drop database by terminating connections
    ```bash
    # Making sure the database exists
    SELECT * from pg_database where datname = 'my_database_name'
    
    # Disallow new connections
    UPDATE pg_database SET datallowconn = 'false' WHERE datname = 'my_database_name';
    ALTER DATABASE my_database_name CONNECTION LIMIT 1;
    
    # Terminate existing connections
    SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'my_database_name';
    
    # Drop database
    DROP DATABASE my_database_name
    ```

5. Remove specific version if multiple versions installed
    ```bash
    # List and verify components to be removed
    dpkg -l | grep postgres
 
    ii  pgdg-keyring                          2018.2                                          all          keyring for apt.postgresql.org
    ii  postgresql-10                         10.12-2.pgdg18.04+1                             amd64        object-relational SQL database, version 10 server
    ii  postgresql-12                         12.2-2.pgdg18.04+1                              amd64        object-relational SQL database, version 12 server
    ii  postgresql-12-postgis-3               3.0.1+dfsg-2.pgdg18.04+1                        amd64        Geographic objects support for PostgreSQL 12
    ii  postgresql-12-postgis-3-scripts       3.0.1+dfsg-2.pgdg18.04+1                        all          Geographic objects support for PostgreSQL 12 -- SQL scripts
    ii  postgresql-client-10                  10.12-2.pgdg18.04+1                             amd64        front-end programs for PostgreSQL 10
    ii  postgresql-client-12                  12.2-2.pgdg18.04+1                              amd64        front-end programs for PostgreSQL 12
    ii  postgresql-client-common              213.pgdg18.04+1                                 all          manager for multiple PostgreSQL client versions
    ii  postgresql-common                     213.pgdg18.04+1                                 all          PostgreSQL database-cluster manager
 
    # Delete pg 12
    sudo apt-get --purge remove postgresql-12 postgresql-12-postgis-3 postgresql-12-postgis-3-scripts postgresql-client-12
    
    # Verify residuals
    dpkg -l | grep postgres
 
    ii  pgdg-keyring                          2018.2                                          all          keyring for apt.postgresql.org
    ii  postgresql-10                         10.12-2.pgdg18.04+1                             amd64        object-relational SQL database, version 10 server
    ii  postgresql-client-10                  10.12-2.pgdg18.04+1                             amd64        front-end programs for PostgreSQL 10
    ii  postgresql-client-common              213.pgdg18.04+1                                 all          manager for multiple PostgreSQL client versions
    ii  postgresql-common                     213.pgdg18.04+1                                 all          PostgreSQL database-cluster manager
    ```

## Monitoring
1. Basic monitoring - [Telegraf](https://docs.influxdata.com/telegraf/v1.13/)
    * Installation -
        * https://docs.influxdata.com/telegraf/v1.13/introduction/installation/
    * Configuration -
        * https://github.com/influxdata/telegraf/tree/master/plugins/inputs/postgresql
        * https://github.com/influxdata/telegraf/tree/master/plugins/inputs/postgresql_extensible
        * https://www.postgresql.org/docs/10/monitoring-stats.html
    * Sample configuration can be found [here](monitoring/telegraf.conf).

2. Performance monitoring - [PgHero](https://github.com/ankane/pghero) 
    * Linux setup guide: https://github.com/ankane/pghero/blob/master/guides/Linux.md
    * A sample pg_hero.yml is available [here](monitoring/sample_pg_hero.yml).

3. Log monitoring - [PgBadger](https://github.com/darold/pgbadger)
    * Installation - 
        ```bash
        sudo apt-get install pgbadger --fix-missing
        which pgbadger
        pgbadger /var/log/postgresql/postgresql-10-main.log
        ```
    * Systemd units for python webserver, pgbadger service and timer are present [here](systemd). 

4. Connection Pooling - [PgBouncer](https://github.com/pgbouncer/pgbouncer)
    * Linux setup guide: https://tech.willandskill.se/how-to-setup-pgbouncer-on-ubuntu-18-04-lts/