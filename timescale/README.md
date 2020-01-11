# Timescale Server Setup
To know more about Timescale, visit https://www.timescale.com/

## Overview

From the official docs -

> TimescaleDB is an open-source time-series database optimized for fast ingest and complex queries. It speaks "full SQL" and is correspondingly easy to use like a traditional relational database, yet scales in ways previously reserved for NoSQL databases.

## Setup

1. Installation & Setup - Master
    * Follow the documentation for single node installation: https://docs.timescale.com/v1.1/getting-started/installation/rhel-centos/installation-yum
    * Follow the documentation for master setup: https://docs.timescale.com/v1.1/getting-started/setup
    
2. Setup Replication - Slave
    * Refer the documentation for getting the context: https://docs.timescale.com/v1.1/tutorials/replication

> *Steps to configure the Primary Database:*

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

> *Steps to configure the Replica Database:*
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


> *Other Settings*
1. Delete wals on master node which are older than n days: [n=5]
    ```bash
    crontab -e
    30 05 1-31/3 * * find /data/postgresql/postgresql/10/main/pg_wal_archive -mtime +3 -delete
    ```
2. Run timescaledb tune to tune your servers as per resources available (run this command on both master as well as replicas, then restart postgresql)
    ```bash
    timescaledb-tune
    ```