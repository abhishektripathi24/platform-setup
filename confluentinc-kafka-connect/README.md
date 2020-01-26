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

3. Update connect-distributed.properties
    ```bash
    vim /opt/confluent/etc/kafka/connect-distributed.properties
       > bootstrap.servers=10.11.18.58:9092,10.11.18.59:9092,10.11.18.60:9092
       > group.id=connect-cluster
       > offset.storage.replication.factor=3
       > config.storage.replication.factor=3
       > status.storage.replication.factor=3
       > plugin.path=share/java
    ```

4. Start the process on each server
    ```bash
    cd /opt/confluent
    ./bin/connect-distributed  etc/kafka/connect-distributed.properties
    ```
    
 5. If you linux distro supports systemd, you can supervise kafka connect process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.
 
 ## References
 * https://docs.confluent.io/current/connect/concepts.html