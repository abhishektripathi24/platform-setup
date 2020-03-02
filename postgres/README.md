# PostgreSQL [Master-Slave Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/postgres/images/postgresql-logo.png" width="200" height="200"/>

To know more about postgresql, visit https://www.postgresql.org/

## Overview

From the official docs -

> PostgreSQL is a powerful, open source object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

## Setup

For master-slave setup, refer following [timescale's documentation](../timescale/README.md).

## Administration

1. User Management - [[pg-docs](https://www.postgresql.org/docs/12/sql-createuser.html), [AWS](https://aws.amazon.com/blogs/database/managing-postgresql-users-and-roles/), [Blog](https://tableplus.com/blog/2018/04/postgresql-how-to-grant-access-to-users.html)]
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
    (REASSIGN | DROP) OWNED BY readonly to postgres;
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
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT ON TABLES TO readonly;
    
    -- Read/write role
    CREATE ROLE readwrite;
    GRANT CONNECT ON DATABASE mydatabase TO readwrite;
    GRANT USAGE, CREATE ON SCHEMA myschema TO readwrite;
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA myschema TO readwrite;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;
    GRANT USAGE ON ALL SEQUENCES IN SCHEMA myschema TO readwrite;
    ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT USAGE ON SEQUENCES TO readwrite;
    
    -- Users creation
    CREATE USER reporting_user1 WITH PASSWORD 'some_secret_passwd';
    CREATE USER reporting_user2 WITH PASSWORD 'some_secret_passwd';
    CREATE USER app_user1 WITH PASSWORD 'some_secret_passwd';
    CREATE USER app_user2 WITH PASSWORD 'some_secret_passwd';
    
    -- Grant privileges to users
    GRANT readonly TO reporting_user1;
    GRANT readonly TO reporting_user2;
    GRANT readwrite TO app_user1;
    GRANT readwrite TO app_user2;
    ``` 

2. Database and Table size
    ```bash
    SELECT pg_size_pretty(pg_database_size('dbname'));
    SELECT pg_size_pretty(pg_total_relation_size('tablename'));
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