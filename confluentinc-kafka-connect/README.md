# Kafka Connect [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/confluentinc-kafka-connect/images/kafka-connect-logo.png" width="500" height="350"/>

To know more about Kafka Connect, visit https://docs.confluent.io/current/connect/index.html

## Overview

From the official docs -

> Kafka Connect, an open source component of Kafka, is a framework for connecting Kafka with external systems such as databases, key-value stores, search indexes, and file systems.
  
> Using Kafka Connect you can use existing connector implementations for common data sources and sinks to move data into and out of Kafka.

## Setup

1. Install Java and set JAVA_HOME
    ```bash
    sudo apt update
    sudo apt install openjdk-8-jdk openjdk-8-jre
    java -version
    sudo vim /etc/environment
    JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
    JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
    source /etc/environment
    echo $JAVA_HOME
    ```

2. Download confluent binary
    ```bash
    cd /opt
    sudo wget https://packages.confluent.io/archive/5.4/confluent-5.4.0-2.12.tar.gz
    sudo tar -xzf confluent-5.4.0-2.12.tar.gz
    sudo mv confluent-5.4.0-2.12 confluent
    ```

3. Update classpath for any kafka connectors. Add following piece of code in connect-distributed after initializing `java_base_dir` variable. 
    ```bash
    vim /opt/confluent/bin/connect-distributed
 
        # Classpath addition for any Kafka Connect connectors
        for library in $java_base_dir/kafka-connect-*; do
        classpath_prefix="$CLASSPATH:"
        if [ "x$CLASSPATH" = "x" ]; then
        classpath_prefix=""
        fi
        CLASSPATH="$classpath_prefix$library/*"
        done
    ```

4. Update connect-distributed.properties
    ```bash
    vim /opt/confluent/etc/kafka/connect-distributed.properties
       > bootstrap.servers=10.11.18.58:9092,10.11.18.59:9092,10.11.18.60:9092
       > group.id=connect-cluster
       > offset.storage.replication.factor=3
       > config.storage.replication.factor=3
       > status.storage.replication.factor=3
       > plugin.path=share/java
    ```

5. Start the process on each server
    ```bash
    cd /opt/confluent
    ./bin/connect-distributed  etc/kafka/connect-distributed.properties
    ```

 6. If you linux distro supports systemd, you can supervise kafka connect process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.
 
 ## Debezium
 
> Debezium is a set of distributed services to capture changes in your databases so that your applications can see those changes and respond to them. Debezium records all row-level changes within each database table in a change event stream, and applications simply read these streams to see the change events in the same order in which they occurred.
 
To know more about Debezium, visit https://debezium.io/documentation/reference/1.0/

1. To add following connectors, extract them inside /opt/confluent/share/java as kafka-connect-{xyz}
    * Debezium MySQL connector - https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.0.0.Final/debezium-connector-mysql-1.0.0.Final-plugin.tar.gz
    * Debezium PostgreSQL connector - https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.0.0.Final/debezium-connector-postgres-1.0.0.Final-plugin.tar.gz 
    * Debezium MongoDB connector - https://repo1.maven.org/maven2/io/debezium/debezium-connector-mongodb/1.0.0.Final/debezium-connector-mongodb-1.0.0.Final-plugin.tar.gz

 ## Usage
 * Rest APIs:
     * List all connectors -
        * curl -X GET <connect_node_ip>:8083/connectors/
     * Create a new connector:
        * [source connector](connectors-config.txt) - curl -X POST <connect_node_ip>:8083/connectors -d '{...}'
        * [sink connector](connectors-config.txt)
     * Get connector -
        * curl -X GET <connect_node_ip>:8083/connectors/<connector-name>
        * curl -X GET <connect_node_ip>:8083/connectors/<connector-name>/config
        * curl -X GET <connect_node_ip>:8083/connectors/<connector-name>/status
     * Pause connector -
        * curl -X PUT <connect_node_ip>:8083/connectors/<connector-name>/pause
     * Resume connector -
        * curl -X PUT <connect_node_ip>:8083/connectors/<connector-name>/resume
     * Restart connector -
        * curl -X POST <connect_node_ip>:8083/connectors/<connector-name>/restart
     * Delete connector -
        * curl -X PUT <connect_node_ip>:8083/connectors/<connector-name>

 ## References
 * https://docs.confluent.io/current/connect/concepts.html
 * https://debezium.io/releases/1.0/
 * https://docs.confluent.io/current/connect/references/restapi.html
 * https://docs.confluent.io/current/connect/managing.html
 * https://docs.confluent.io/current/connect/managing/connectors.html
 * https://docs.confluent.io/current/connect/transforms/index.html