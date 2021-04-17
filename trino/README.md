# Trino [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/trino/images/trino-logo.png" width="420" height="200"/>

To know more about Trino, visit https://trino.io/

## Overview
From the official docs -

> Trino is a fast distributed SQL query engine for big data analytics that helps you explore your data universe.

## Setup
Installation of `Trino 355` on `Ubuntu 18.04.3 LTS` - [ref](https://trino.io/docs/current/installation.html)

1. Configure Linux OS
    ```bash
   # Open file descriptors: The number of open file descriptors needed for a particular Trino instance scales as roughly the number of machines in the cluster, times some factor depending on the workload.
   /etc/security/limits.conf:
   
   trino soft nofile 131072
   trino hard nofile 131072  
   ```
   
2. Install Java11 (Recommended: Azul Zulu as the JDK for Trino) - [ref](https://www.azul.com/downloads/zulu-community/?version=java-11-lts&os=ubuntu&architecture=x86-64-bit&package=jdk) 
    ```bash
    mkdir -p /usr/lib/jvm && cd /usr/lib/jvm
    wget https://cdn.azul.com/zulu/bin/zulu11.45.27-ca-jdk11.0.10-linux_x64.tar.gz
    tar -xzf zulu11.45.27-ca-jdk11.0.10-linux_x64.tar.gz
    ```

3. Install python (required by the bin/launcher script only)
    ```bash
    any version 2.6.x, 2.7.x, or 3.x
    ``` 

4. Download the Trino server tarball and unpack it - [ref](https://repo1.maven.org/maven2/io/trino/trino-server/355/trino-server-355.tar.gz)
    ```bash
    cd /opt
    wget https://repo1.maven.org/maven2/io/trino/trino-server/355/trino-server-355.tar.gz
    tar -xzf trino-server-355.tar.gz
    mv trino-server-355 trino
    cd trino
    ```

5. Configure Trino through following properties - [ref](https://trino.io/docs/current/installation/deployment.html)
    * Create `etc` directory inside Trino on both coordinator and worker
        ```bash
        cd /opt/trino
        mkdir etc
       ```
    * Create `etc/node.properties` for node specific properties (Prefer to use an externally mounted data directory at `/data`)
        * At Coordinator Node:
            ```properties
            node.environment=production
            node.id=trino-prod-coordinator
            node.data-dir=/data/trino/data
            ```
        * At Worker Node:
            ```properties
            node.environment=production
            node.id=trino-prod-worker-1
            node.data-dir=/data/trino/data
            ``` 
    * Create `etc/jvm.config` for jvm specific properties on both coordinator and worker
        ```properties
        -server
        -Xmx16G
        -XX:-UseBiasedLocking
        -XX:+UseG1GC
        -XX:G1HeapRegionSize=32M
        -XX:+ExplicitGCInvokesConcurrent
        -XX:+ExitOnOutOfMemoryError
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:ReservedCodeCacheSize=512M
        -XX:PerMethodRecompilationCutoff=10000
        -XX:PerBytecodeRecompilationCutoff=10000
        -Djdk.attach.allowAttachSelf=true
        -Djdk.nio.maxCachedBufferSize=2000000
        ```
    * Create `etc/config.properties` for configuring trino server
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
    * Create `etc/log.properties` for configuring log levels on both coordinator and worker
        ```properties
        io.trino=INFO
        ```
    * Create directory `etc/catalog` for adding `catalogs` on both coordinator and worker
        ```bash
        mkdir catalog
        ``` 
    * Add catalogs as follows on both coordinator and worker - [ref](https://trino.io/docs/current/connector.html)
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

6. Start the process on both coordinator and worker
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
        /data/trino/data/var/log/launcher.log
        /data/trino/data/var/log/server.log
        /data/trino/data/var/log/http-request.log
        ```

7. Trino CLI for terminal-based interactive shell on coordinator
    * Download `trino-cli-355-executable.jar` and rename it to trino - [ref](https://repo1.maven.org/maven2/io/trino/trino-cli/355/trino-cli-355-executable.jar) 
        ```bash
        wget https://repo1.maven.org/maven2/io/trino/trino-cli/355/trino-cli-355-executable.jar
        mv trino-cli-355-executable.jar trino
        chmod +x trino
        ``` 
    * Login to shell
        ```bash
        ./trino --server localhost:8080 --catalog <name> --schema default
        ```
    * Verify connected nodes via `trino-cli`
        ```bash
        select * from system.runtime.nodes;
        ```
    * Verify available catalogs via `trino-cli`
        ```bash
        show catalogs; 
        ```
    * Verify runtime query stats via Web Interface available at `http://localhost:8080`
       
8. If you linux distro supports systemd, you can supervise trino process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## Security
1. Securing client access to the cluster using load balancer or proxy to terminate HTTPS - [ref](https://trino.io/docs/current/security/overview.html)
    * Update `etc/config.properties` to include following
        ```properties
        http-server.process-forwarded=true
        ```
2. Authentication using Password File - [ref](https://trino.io/docs/current/security/password-file.html)
    * Update `etc/config.properties`
        ```properties
        http-server.authentication.type=PASSWORD
        ```
    * Create `etc/password-authenticator.properties`
        ```properties
        password-authenticator.name=file
        file.password-file=/path/to/password.db
        file.refresh-period=1m
        file.auth-token-cache.max-size=1000
        ```
    * Install `htpasswd` utility
        ```bash
        sudo apt-get install apache2-utils
        ```
    * Create `password.db` with the users
        ```bash
        # Create file
        touch /path/to/password.db
        
        # Create user with password
        htpasswd -B -C 10 /path/to/password.db test_user
        
        # Verify user password
        htpasswd -v /path/to/password.db test_user
        ```

3. Authorization and Access Control
    * User Groups - [ref](https://trino.io/docs/current/security/group-file.html)
    
        * Create `etc/group-provider.properties`
            ```properties
            group-provider.name=file
            file.group-file=/path/to/group.txt
            ```
        * Create `/path/to/group.txt`
            ```text
            group_name_1:user_1,user_2,user_3
            group_name_2:user_4,user_5
            ```
    * Access Control for users and groups - [ref](https://trino.io/docs/current/security/file-system-access-control.html)
    
        * Create `etc/access-control.properties`
            ```properties
            access-control.name=file
            security.config-file=/path/to/access_control_rules.json
            security.refresh-period=1s
            ```
        * Create `/path/to/access_control_rules.json`
            ```json
            {
                "queries": [
                    {
                        "user": "group_name_admin",
                        "allow": [
                            "execute",
                            "kill",
                            "view"
                        ]
                    },
                    {
                        "allow": [
                            "execute"
                        ]
                    }
                ],
                "catalogs": [
                    {
                        "group": "group_name_1",
                        "catalog": "(jmx|system|catalog_name_1|catalog_name_2)",
                        "allow": "all"
                    },
                    {
                        "group": "group_name_2",
                        "catalog": "(catalog_name_1|catalog_name_2)",
                        "allow": "read-only"
                    },
                    {
                        "catalog": "system",
                        "allow": "none"
                    }
                ]
            }
            ```    

## Monitoring
* Monitoring using metrics exposed by [JMX Exporter](https://github.com/prometheus/jmx_exporter):
    * Download the jar
        ```bash
         mkdir /opt/jmx-exporter && cd /opt/jmx-exporter 
         wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.15.0/jmx_prometheus_javaagent-0.15.0.jar
         touch /opt/jmx-exporter/exporter_config.yaml 
        ```
    * Update `trino/etc/jvm.config` to include the `javaagent`
        ```bash
        -javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent-0.15.0.jar=9090:/opt/jmx-exporter/exporter_config.yaml 
        ```
    * Restart the trino server.
    * Metrics will now be accessible at `http://localhost:9090/metrics`
    * A sample grafana dashboard is available [here](monitoring).
    * Dashboard constituents version -
        * Grafana - `Grafana v5.3.4 (69630b9)`
        * Telegraf - `Telegraf 1.13.1 (git: HEAD 0c175724)`

## References
* https://trino.io/docs/current/installation/deployment.html