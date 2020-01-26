# Schema Registry [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/confluentinc-schema-registry/images/schema-registry-logo.png" width="600" height="300"/>

To know more about Schema Registry, visit https://docs.confluent.io/current/schema-registry/index.html

## Overview

From the official docs -

> Confluent Schema Registry provides a serving layer for your metadata. It provides a RESTful interface for storing and retrieving Apache Avro® schemas. It stores a versioned history of all schemas based on a specified subject name strategy, provides multiple compatibility settings and allows evolution of schemas according to the configured compatibility settings and expanded Avro support. It provides serializers that plug into Apache Kafka® clients that handle schema storage and retrieval for Kafka messages that are sent in the Avro format.
  
> Schema Registry lives outside of and separately from your Kafka brokers. Your producers and consumers still talk to Kafka to publish and read data (messages) to topics. Concurrently, they can also talk to Schema Registry to send and retrieve schemas that describe the data models for the messages.

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

3. Update schema-registry.properties
    ```bash
    vim /opt/confluent/etc/schema-registry/schema-registry.properties
       > kafkastore.bootstrap.servers=PLAINTEXT://10.11.18.58:9092,PLAINTEXT://10.11.18.59:9092,PLAINTEXT://10.11.18.60:9092
    ```

4. Start the process on each server
    ```bash
    cd /opt/confluent
    ./bin/schema-registry-start etc/schema-registry/schema-registry.properties
    ```
 
 5. If you linux distro supports systemd, you can supervise kafka process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.
 
 ## References
 * https://docs.confluent.io/current/schema-registry/schema_registry_tutorial.html