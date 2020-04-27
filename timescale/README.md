# Timescale [Master-Slave Setup]
![](images/timescale-logo.png)

To know more about Timescale, visit https://www.timescale.com/

## Overview
From the official docs -

> TimescaleDB is an open-source time-series database optimized for fast ingest and complex queries. It speaks "full SQL" and is correspondingly easy to use like a traditional relational database, yet scales in ways previously reserved for NoSQL databases.

## Setup
Installation of `Timescale 1.2.2 over Postgres 10` on `Ubuntu 18.04.3 LTS` - [ref](https://docs.timescale.com/v1.1/getting-started/installation/ubuntu/installation-apt-ubuntu)

1. Setup - Master
    * Follow the documentation for single node installation: https://docs.timescale.com/v1.1/getting-started/installation/ubuntu/installation-apt-ubuntu
    * Follow the documentation for master setup: https://docs.timescale.com/v1.1/getting-started/setup

2. Setup streaming replication - Slave: https://docs.timescale.com/v1.1/tutorials/replication
    * Steps to configure the *Primary Database* -
        1. Configure the Primary Database
            ```postgresql
            SET password_encryption = 'scram-sha-256';
            CREATE ROLE repuser WITH REPLICATION PASSWORD 'password' LOGIN;
            ``` 
        2. Configure Replication parameters - Async Replication with 1 Replica
            ```bash
            vim /etc/postgresql/10/main/postgresql.conf
            
                listen_address = '0.0.0.0'
                wal_level = replica
                max_wal_senders = 10
                max_replication_slots = 2
                max_wal_size = 1024
                min_wal_size = 80
                wal_keep_segments = 32
                wal_receiver_timeout = 30000
                wal_sender_timeout = 30000
                archive_mode = on
                archive_command = 'cp %p /data/postgresql/postgresql/10/main/pg_wal_archive/%f'
            ```
        3. Restart postgreSQL
            ```bash
            sudo systemctl restart postgresql # or
            sudo service postgresql restart
            ```
        4. Create replication slot for each replica through psql shell
            ```postgresql
            SELECT * FROM pg_create_physical_replication_slot('replica_1_slot');
            ```
        5. Configure Host Based Authentication
            ```bash
            vim /etc/postgresql/10/main/pg_hba.conf
            >   host replication repuser <hostname> scram-sha-256
            vim /etc/hosts
            >  <ip> <hostname>
            ```

    * Steps to configure the *Replica Database*:
        1. Create a base backup on replica
            ```bash
            rm -rf <DATA_DIRECTORY>/*
            pg_basebackup -h <PRIMARY_IP> -D <DATA_DIRECTORY> -U repuser -vP -W
            ```
        2. Create a recovery.conf file in data directory
            ```bash
            touch <DATA_DIRECTORY>/recovery.conf
            chmod 0600 <DATA_DIRECTORY>/recovery.conf
            ```
        3. Replication and Recovery Settings
            ```bash
            vim /data/postgresql/postgresql/10/main/recovery.conf
            >  standby_mode = on # Ensures that the replica continues to fetch WAL records from the primary
            >  primary_conninfo = 'host=abc.com port=5432 user=repuser password=password application_name=r1'
            >  primary_slot_name = 'replica_1_slot' # Name of the replication slot we created on the master
            
            vim /etc/postgresql/10/main/postgresql.conf
            >  hot_standby = on
            >  wal_level = replica
            >  max_wal_senders = 2
            >  max_replication_slots = 2
            >  synchronous_commit = off
            ```
        4. Restart PostgreSQL
        
## Misc
1. Delete wals on master node which are older than n days: [n=5]
    ```bash
    crontab -e
    30 05 1-31/3 * * find /data/postgresql/postgresql/10/main/pg_wal_archive -mtime +3 -delete
    ```
2. Run [timescaledb-tune](https://github.com/timescale/timescaledb-tune) to tune your servers as per resources available (run this command on both master as well as replicas, then restart postgresql)
    ```bash
    timescaledb-tune
    ```
3. Sample custom `postgresql.conf` on a 2cores-4GiB machine: 
    ```bash
    #------------------------------------------------------------------------------
    # CUSTOMIZED OPTIONS
    #------------------------------------------------------------------------------
    
    # Add settings for extensions here
    
    # Connection settings
    listen_addresses = '*'
    
    # Directory settings
    data_directory = '/data/postgresql/postgresql/10/main'
    
    # Replication settings (master)
    max_wal_senders = 10
    max_replication_slots = 4
    wal_keep_segments = 32
    wal_receiver_timeout = 30000
    wal_sender_timeout = 30000
    
    # Memory settings
    shared_buffers = 950246kB
    work_mem = 4743kB
    maintenance_work_mem = 475123kB
    
    # Query settings
    effective_cache_size = 2783MB
    default_statistics_target = 500
    random_page_cost = 1.1
    statement_timeout = 600000 # 10 mins
    log_min_duration_statement = 10000 # 10 secs
    #shared_preload_libraries = 'pg_stat_statements'
    #pg_stat_statements.track = all
    
    # Parallelism settings
    max_worker_processes = 13
    max_parallel_workers_per_gather = 1
    max_parallel_workers = 2
    dynamic_shared_memory_type = posix
    timescaledb.max_background_workers = 8
    
    # WAL settings
    wal_level = replica
    min_wal_size = 4GB
    max_wal_size = 8GB
    wal_buffers = 16MB
    synchronous_commit = off
    checkpoint_completion_target = 0.9
    archive_mode = on
    archive_command = 'cp %p /data/postgresql/postgresql/10/main/pg_wal_archive/%f'
    #archive_command = 'rsync -a %p barman@<barman-ip>:/data/timescaledb-master/incoming/%f'
    
    # Miscellaneous settings
    max_connections = 200
    effective_io_concurrency = 200
    ```

4. Sample `pg_hba.conf` for remote login and streaming replication:
    ```bash
    # TYPE     DATABASE        USER            ADDRESS METHOD            AUTH_METHOD
    host       replication     repuser         <replica-server-ip>       scram-sha-256
    host       all             all             all                       md5
    ``` 