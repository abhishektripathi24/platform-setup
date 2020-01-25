# Zookeeper [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/apache-zookeeper/images/zookeeper-logo.png" width="500" height="250"/>

To know more about Zookeeper, visit https://zookeeper.apache.org/

## Overview

From the official docs -

> Apache ZooKeeper is an effort to develop and maintain an open-source server which enables highly reliable distributed coordination.

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
    
2. Download Zookeeper Binary
    ```bash
    cd /opt
    wget http://www-eu.apache.org/dist/zookeeper/stable/apache-zookeeper-3.5.6-bin.tar.gz
    tar -xzf apache-zookeeper-3.5.6-bin.tar.gz
    mv apache-zookeeper-3.5.6-bin.tar.gz zookeeper
    cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg    
    ```
    
3. Update zoo.cfg (the following steps are for a cluster of 3 nodes)
    ```bash
    vim /opt/zookeeper/conf/zoo.cfg
        > dataDir=/var/lib/zookeeper
        > server.1=10.11.18.58:2888:3888
        > server.2=10.11.18.59:2888:3888
        > server.3=10.11.18.60:2888:3888
        > 4lw.commands.whitelist=srvr,stat,wchc,dump,crst,srst,envi,conf,telnet,wchs,wchp,dirs,cons,mntr,isro,ruok,gtmk,stmk
    ```

4. Provide unique ids on each zookeeper node
    ```bash
    sudo mkdir /var/lib/zookeeper
    sudo touch /var/lib/zookeeper/myid
    
    # on zookeeper server 1
    sudo sh -c "echo '1' > /var/lib/zookeeper/myid"
    # on zookeeper server 2
    sudo sh -c "echo '2' > /var/lib/zookeeper/myid"
    # on zookeeper server 3
    sudo sh -c "echo '3' > /var/lib/zookeeper/myid"
    ``` 
    
5. Start the process on each server
    ```bash
    cd /opt/zookeeper
    ./bin/zkServer.sh start-foreground # to run in foreground
    ./bin/zkServer.sh start # to run in background
    ```

6. Test the setup on each node
    ```bash
    echo "ruok" | nc localhost 2181
    > imok
    ```
7. If you linux distro supports systemd, you can supervise zookeeper process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.
###### NOTE: You should always keep your data on an externally mounted volume instead of the root volume if you are running the setup on cloud. This is to ensure that your data stays intact in case the VM become unresponsive. So consider you have mounted your external volume on /data, then update dataDir in zoo.cfg as follows: dataDir=/data/zookeeper 