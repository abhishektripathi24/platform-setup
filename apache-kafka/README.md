# Kafka [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/apache-kafka/images/kafka-logo.png" width="600" height="180"/>

To know more about Kafka, visit https://kafka.apache.org/

## Overview

From the official docs -

> Kafka is generally used for two broad classes of applications:

> a) Building real-time streaming data pipelines that reliably get data between systems or applications

> b) Building real-time streaming applications that transform or react to the streams of data

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

2. Download kafka binary
    ```bash
    cd /opt
    sudo wget http://mirrors.estointernet.in/apache/kafka/2.4.0/kafka_2.12-2.4.0.tgz
    sudo tar -xzf kafka_2.12-2.4.0.tgz
    sudo mv kafka_2.12-2.4.0 kafka
    ```

3. Update zookeeper.properties and server.properties (assuming the data directory is mounted at /data - as per the [NOTE](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-zookeeper)  in zk setup)
    ```bash
    vim /opt/kafka/config/zookeeper.properties
       > dataDir=/data/zookeeper

    sudo vim /opt/kafka/config/server.properties
       > broker.id=0 # this should be unique for each broker node
       > advertised.listeners=PLAINTEXT://10.11.18.59:9092 # ip-address/url of this server itself
       > log.dirs=/data/kafka/logs
       > num.partitions=2
       > num.recovery.threads.per.data.dir=3
       > offsets.topic.replication.factor=3
       > transaction.state.log.replication.factor=3
       > log.flush.interval.messages=10000
       > log.flush.interval.ms=1000
       > log.retention.hours=24
       > log.retention.bytes=161061273600
       > zookeeper.connect=<IP/URL_of_zk>:2181/kafka_cluster
    ```

4. Start the process on each server
    ```bash
    cd /opt/kafka
    ./bin/kafka-server-start.sh config/server.properties
    ```

5. Test the setup via zookeeper node (as the brokers get registered on zookeeper). Sample output for a 3 node kafka cluster
    ```bash
    echo dump | nc <zookeeper_ip> 2181 | grep brokers
    > /kafka_cluster/brokers/ids/0
    > /kafka_cluster/brokers/ids/1
    > /kafka_cluster/brokers/ids/2
    ```
 
 6. If you linux distro supports systemd, you can supervise kafka process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.
 
 ## References
 * https://kafka.apache.org/quickstart