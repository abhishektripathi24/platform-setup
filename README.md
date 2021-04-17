# Platform Setup
This repo contains guidelines and steps for setting up in-house production infrastructure and cloud services of open-source technologies from scratch.

## Data Engineering Stack 
* CDC Pipelines
    * High-performance coordination service for distributed applications:
        * [Apache Zookeeper](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-zookeeper)
    * Distributed event streaming platform: 
        * [Apache Kafka](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-kafka)
    * Distributed framework to stream data into and out of Apache kafka:
        * [Confluent Inc. Kafka Connect](https://github.com/abhishektripathi24/platform-setup/tree/master/confluentinc-kafka-connect)
    * Distributed registry to store kafka-payload's schemas:  
        * [Confluent Inc. Schema Registry](https://github.com/abhishektripathi24/platform-setup/tree/master/confluentinc-schema-registry)    
    
* Databases
    * Relational:
        * [PostgreSQL](https://github.com/abhishektripathi24/platform-setup/tree/master/postgres)
        * [Timescale](https://github.com/abhishektripathi24/platform-setup/tree/master/timescale) 
        * [Mysql](https://github.com/abhishektripathi24/platform-setup/tree/master/mysql)
    * Document:
        * [MongoDB](https://github.com/abhishektripathi24/platform-setup/tree/master/mongo)
    * Key-value:
        * [Redis](https://github.com/abhishektripathi24/platform-setup/tree/master/redis) 
    * Graph:
        * [Neo4j](https://github.com/abhishektripathi24/platform-setup/tree/master/neo4j)
    * Time Series:
        * [Prometheus](https://github.com/abhishektripathi24/platform-setup/tree/master/prometheus) (NoSQL)
        * [Timescale](https://github.com/abhishektripathi24/platform-setup/tree/master/timescale) (SQL)
    
* Distributed Workflow Management
    * [Apache Airflow](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-airflow)
    
* Big Data
    * Distributed SQL Query Engine on any data storage:
        * [Prestodb](https://github.com/abhishektripathi24/platform-setup/tree/master/prestodb)
        * [PrestoSQL/Trino](https://github.com/abhishektripathi24/platform-setup/tree/master/trino)
    * Distributed & Resilient Data Processing framework:
        * [Apache Spark](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-spark)
    * SQL on HDFS:
        * [Apache Hive]()
    
* Search Engines
    * [Elasticsearch](https://github.com/abhishektripathi24/platform-setup/tree/master/elasticsearch)
    
* Centralized Logging
    * [Elastic Stack](https://github.com/abhishektripathi24/platform-setup/tree/master/elastic-stack)
        * Filebeat
        * Elasticsearch-Ingest-Pipeline
        * Kibana
    
* Business Intelligence
    * [Apache Superset](https://github.com/abhishektripathi24/platform-setup/tree/master/apache-superset)

## Platform Engineering Stack

* Container Orchestration
    * [Kubernetes](https://github.com/abhishektripathi24/platform-setup/tree/master/kubernetes)
    
* Service Discovery, Health Checking & Configuration
    * [Consul](https://github.com/abhishektripathi24/platform-setup/tree/master/consul)

* Service Monitoring 
    * [Grafana](https://github.com/abhishektripathi24/platform-setup/tree/master/grafana)
    * [Prometheus](https://github.com/abhishektripathi24/platform-setup/tree/master/prometheus)

* SSL Certs
    * [Let's Encrypt](https://github.com/abhishektripathi24/platform-setup/tree/master/letsencrypt/)

* Load balancing & Reverse Proxying
    * [NGINX](https://github.com/abhishektripathi24/platform-setup/tree/master/nginx/)
    
* VPN
    * [OpenVPN](https://github.com/abhishektripathi24/platform-setup/tree/master/vpn/)
    
* Linux
    * [Linux Utilities](https://github.com/abhishektripathi24/platform-setup/tree/master/linux/)

...

![](logos.png)