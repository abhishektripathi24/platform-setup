# Prestodb [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/prestodb/images/presto-logo.png" width="400" height="200"/>

To know more about Presto, visit https://prestodb.io/

## Overview
From the official docs -

> Presto is a distributed SQL query engine designed to query large data sets distributed over one or more heterogeneous data sources.

## Setup
Installation of `Presto 0.250` on `Ubuntu 18.04.3 LTS` - [ref](https://prestodb.io/docs/current/installation/deployment.html)

1. Install Java8
    ```bash
    sudo apt update
    sudo apt install openjdk-8-jdk openjdk-8-jre
    java -version
    ```

2. Install python (required by the bin/launcher script only)
    ```bash
    any version 2.6.x, 2.7.x, or 3.x
    ``` 

3. Download the Presto server tarball and unpack it - [ref](https://prestodb.io/download.html)
    ```bash
    cd /opt
    wget https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.250/presto-server-0.250.tar.gz
    tar -xzf presto-server-0.250.tar.gz
    mv presto-server-0.250 presto
    cd presto
    ```

4. Configure presto through following properties - [ref](https://prestodb.io/docs/current/installation/deployment.html)
    * Create `etc` directory inside presto
        ```bash
        cd /opt/presto
        mkdir etc
       ```
    * Create `etc/node.properties` for node specific properties (Prefer to use an externally mounted data directory at `/data`)
        * At Coordinator Node:
            ```properties
            node.environment=production
            node.id=presto-prod-coordinator
            node.data-dir=/data/presto/data
            ```
        * At Worker Node:
            ```properties
            node.environment=production
            node.id=presto-prod-worker-1
            node.data-dir=/data/presto/data
            ``` 
    * Create `etc/jvm.config` for jvm specific properties
        ```properties
        -server
        -Xmx16G
        -XX:+UseG1GC
        -XX:G1HeapRegionSize=32M
        -XX:+UseGCOverheadLimit
        -XX:+ExplicitGCInvokesConcurrent
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:+ExitOnOutOfMemoryError
        ```
    * Create `etc/config.properties` for configuring presto server (coordinator/worker)
        * At Coordinator Node:
            ```properties
            coordinator=true
            node-scheduler.include-coordinator=false
            http-server.http.port=8080
            query.max-memory=50GB
            query.max-memory-per-node=1GB
            query.max-total-memory-per-node=2GB
            discovery-server.enabled=true
            discovery.uri=http://localhost:8080
            ```
        * At Worker Node:
            ```properties
            coordinator=false
            http-server.http.port=8080
            query.max-memory=50GB
            query.max-memory-per-node=1GB
            query.max-total-memory-per-node=2GB
            discovery.uri=http://coordinator-node-host-or-ip:8080
            ```
    * Create `etc/log.properties` for configuring log levels
        ```properties
        com.facebook.presto=INFO
        ```
    * Create directory `etc/catalog` for adding `catalogs`
        ```bash
        mkdir catalog
        ``` 
    * Add catalogs as follows - [ref](https://prestodb.io/docs/current/connector.html)
        * Create `etc/catalog/jmx.properties` for jmx catalog
            ```properties
            connector.name=jmx
            ```
        * Create `etc/catalog/mysql.properties` for mysql catalog
            ```properties
            connector.name=mysql
            connection-url=jdbc:mysql://example.net:3306
            connection-user=root
            connection-password=secret
            ```
        * Create `etc/catalog/postgresql.properties` for postgresql catalog
            ```properties
            connector.name=postgresql
            connection-url=jdbc:postgresql://example.net:5432/database
            connection-user=root
            connection-password=secret
            ```

5. Start the process on each server
    * Foreground
        ```bash
        bin/launcher start
        ```
    * Daemon mode
        ```bash
        bin/launcher run
        ```
    * Verify logs at
        ```bash
        /data/presto/data/var/log/launcher.log
        /data/presto/data/var/log/server.log
        /data/presto/data/var/log/http-request.log
        ```

6. Presto CLI for terminal-based interactive shell
    * Download `presto-cli.jar` and rename it to presto - [ref](https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.250/presto-cli-0.250-executable.jar) 
        ```bash
        wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.250/presto-cli-0.250-executable.jar
        mv presto-cli-0.250-executable.jar presto
        chmod +x presto
        ``` 
    * Login to shell
        ```bash
        ./presto --server localhost:8080 --catalog <name> --schema default
        ```
    * Verify connected nodes via `presto-cli`
        ```bash
        select * from system.runtime.nodes;
        ```
    * Verify available catalogs via `presto-cli`
        ```bash
        show catalogs; 
        ```
    * Verify runtime query stats via Web Interface available at `http://localhost:8080`
       
7. If you linux distro supports systemd, you can supervise presto process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## Monitoring
* Monitoring using metrics exposed by [JMX Exporter](https://github.com/prometheus/jmx_exporter):
    * Download the jar
        ```bash
         mkdir /opt/jmx-exporter && cd /opt/jmx-exporter 
         wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.15.0/jmx_prometheus_javaagent-0.15.0.jar
         touch /opt/jmx-exporter/exporter_config.yaml 
        ```
    * Update `presto/etc/jvm.config` to include the `javaagent`
        ```bash
        -javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent-0.15.0.jar=9090:/opt/jmx-exporter/exporter_config.yaml 
        ```
    * Restart the presto server.
    * Metrics will now be accessible at `http://localhost:9090/metrics`
    * A sample grafana dashboard is available [here](monitoring).
    * Dashboard constituents version -
        * Grafana - `Grafana v5.3.4 (69630b9)`
        * Telegraf - `Telegraf 1.13.1 (git: HEAD 0c175724)`

## References
* https://prestodb.io/docs/current/installation/deployment.html