# Elasticsearch [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/elasticsearch/images/elasticsearch-logo.png" width="400" height="110"/>

To know more about Elasticsearch, visit https://www.elastic.co/webinars/getting-started-elasticsearch

## Overview
From the official docs of elasticsearch -

> Elasticsearch is the distributed search and analytics engine at the heart of the Elastic Stack.

## Setup
Installation of `Elasticsearch 7.6` on `Ubuntu 18.04.3 LTS` - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

1. Install Java
 
    **NOTE:** Elasticsearch binary comes with a packaged java distribution so explicit java installation is not required.

2. Download and extract standalone linux binary - [ref](https://www.elastic.co/guide/en/elastic-stack-get-started/7.6/get-started-elastic-stack.html#install-elasticsearch)
    ```bash
    cd /opt
    sudo curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.1-linux-x86_64.tar.gz
    sudo tar -zxvf elasticsearch-7.6.1-linux-x86_64.tar.gz
    sudo mv elasticsearch-7.6.1 elasticsearch
    cd elasticsearch
    ```
   
3. (Optional if not installed via step 2) Update apt repository and install elasticsearch
    ```bash
    sudo apt install openjdk-8-jdk -y
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    sudo sh -c 'echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" > /etc/apt/sources.list.d/elastic-7.x.list'
    sudo apt update
    sudo apt install elasticsearch -y 
    ```

4. Configure OS: These settings will prevent errors during starting elasticsearch process - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/master/tune-for-indexing-speed.html#_use_auto_generated_ids)
    ```bash
    # Max map count
    sudo sysctl vm.max_map_count=262144 # Temporary
    echo vm.max_map_count=262144 >> /etc/sysctl.conf # Permanent
   
    # Swappiness
    sudo sysctl vm.swappiness=1 # Temporary
    echo vm.swappiness=1 >> /etc/sysctl.conf # Permanent
   
    # OOM task kill
    sudo sysctl vm.oom_kill_allocating_task=1 # Temporary
    echo vm.oom_kill_allocating_task=1 >> /etc/sysctl.conf # Permanent
   
    # Dirty data writebacks (optional and needed to be played around as per the instance size)
    vm.dirty_writeback_centisecs = 500
    vm.dirty_ratio = 60
    vm.dirty_background_ratio = 5
    ``` 
   
5. Update `config/elasticsearch.yml` to bootstrap the cluster using default discovery method (the following properties are for a cluster of 3 nodes) - [initial-bootstrap-ref](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-discovery-bootstrap-cluster.html), [discovery-settings-ref](https://www.elastic.co/guide/en/elasticsearch/reference/master/discovery-settings.html)
    ```yaml
    # Common settings
    cluster.name: es-cluster
    node.name: node-1
    path.data: /data/data
    path.logs: /data/logs
    network.host: 0.0.0.0
   
    # Discovery settings
    discovery.seed_hosts: ["<node-1-ip-or-hostname>", "<node-2-ip-or-hostname>", "<node-3-ip-or-hostname>"]
    cluster.initial_master_nodes: ["node-1", "node-2", "node-3"] # list of node.name of master eligible nodes 
    ```
    **NOTE:** The above configuration enables all 3 nodes as Master, Data, Ingest, ML, Coordination and Voting eligible nodes. If you want to have dedicated nodes for each of them, following the [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html).

6. The discovery process can also be carried out using cloud plugins - [ref](https://www.elastic.co/guide/en/elasticsearch/plugins/current/discovery.html) 
   * Install the plugin - [ref](https://www.elastic.co/guide/en/elasticsearch/plugins/current/discovery-ec2.html#discovery-ec2-remove)
       ```bash
       ./bin/elasticsearch-plugin install discovery-ec2
       ```
   * Update plugin related config in `config/elasticsearch.yml` - [ref](https://www.elastic.co/guide/en/elasticsearch/plugins/master/discovery-ec2-usage.html) 
       ```yaml
        # e.g. using AWS discovery plugin 
        discovery.zen.minimum_master_nodes: 2
        discovery.zen.hosts_provider: ec2
        discovery.ec2.endpoint: ec2.ap-southeast-1.amazonaws.com
        discovery.ec2.tag.es_discovery: es-master-cluster
        discovery.ec2.availability_zones: ap-southeast-1a,ap-southeast-1b,ap-southeast-1c
        ```
   * Provide following tag to the instances
       ```bash
       es_discovery: es-master-cluster 
       ```
   * Create IAM role with the following policy and attach the role with the instances - [ref](https://www.elastic.co/guide/en/elasticsearch/plugins/master/discovery-ec2-usage.html)
        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": "ec2:DescribeInstances",
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ]
        }
        ```
   * Provide aws credentials if you can't create IAM role ([ref](https://www.elastic.co/guide/en/elasticsearch/plugins/6.0/_settings.html)) and verify from the keystore ([ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-keystore.html)). 
   
7. Update `config/jvm.yml` - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html)
    ```yaml
    -Xms2g
    -Xmx2g
   ```

8. Start the process on each server
    ```bash
    ./bin/elasticsearch
    ``` 
    
9. If your linux distro supports systemd, you can supervise elasticsearch-server process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## Misc
1. Data migration from another elasticsearch cluster - [ref](https://www.elastic.co/guide/en/cloud/current/ec-migrate-data.html)
    * Using `remote reindexing` - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/reindex-upgrade-remote.html#reindex-upgrade-remote)
        * Update `config/elasticsearch.yml` to allow existing cluster
           ```bash
            reindex.remote.whitelist: "<old-server-ip-1>:9200, <old-server-ip-2>:9200, <old-server-ip-3>:9200"
            ```
        * Reindex API - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)
            ```http request
            curl -XPOST localhost:9200/_reindex?pretty -d'{
              "source": {
                "remote": {
                  "host": "http://existing-cluster-ip:9200"
                },
                "index": "index-name"
              },
              "dest": {
                "index": "index-name"
              }
            }'
            ```
        * To run the above api in async mode, provide query param `wait_for_completion=false` and verify the progress of the task created above in async api - [ref](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html)
            ```http request
            curl 'localhost:9200/_tasks/oTUltX4IQMOUUVeiohTt8A:12345'
            ```
        * Verify the newly crated index
            ```http request
            curl 'localhost:9200/_cat/indices?v'
            curl 'localhost:9200/INDEX-NAME/_search?pretty'
            ```

2. To configure a single node elasticsearch cluster and access it from outside, update `config/elasticsearch.yml` with following discovery mechanism
    ```bash
   discovery.type: single-node 
   ```

## Administration
1. List nodes - `curl localhost:9200/_cat/nodes?v` 
2. List indices - `curl localhost:9200/_cat/indices?v`
3. Create default template for 2 shards and 0 replicas
    ```http request
    curl -XPUT localhost:9200/_template/default \
    -d '{
      "default": {
        "order": -1,
        "index_patterns": [
          "*"
        ],
        "settings": {
          "index": {
            "number_of_shards": "2",
            "number_of_replicas": "0"
          }
        },
        "mappings": {
          
        },
        "aliases": {
          
        }
      }
    }'
    ```
4. Get settings and mappings of an index - `curl localhost:9200/<index-name>`
5. Get and update index settings
    ```http request
    curl localhost:9200/<index-name>/_settings
    
    e.g. update replica count
    curl -XPUT localhost:9200/<index-name>/_settings \
    -d '{
      "index": {
        "number_of_replicas": "2"
      }
    }'
    ```
6. Cluster settings: `_cluster/settings?include_defaults=true&flat_settings=true`
7. curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_cluster/settings -d '{ "transient": { "cluster.routing.allocation.disk.threshold_enabled": false } }'
8. curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'

## References
* Architectures - Hot-warm-cold
    * https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management
    * https://www.elastic.co/blog/hot-warm-architecture-in-elasticsearch-5-x
* Best practices
    * https://www.elastic.co/pdf/architecture-best-practices.pdf
    * https://thoughts.t37.net/designing-the-perfect-elasticsearch-cluster-the-almost-definitive-guide-e614eabc1a87#e70b
* Sharding 
    * https://stackoverflow.com/questions/53214628/elasticsearch-how-does-sharding-affect-indexing-performance/53216210#53216210
* Joins
    * https://blog.mimacom.com/parent-child-elasticsearch/
* Cheatsheet
    * https://thoughts.t37.net/an-elasticsearch-cheat-sheet-9b92c9211d7b
* Consistency & Replication
    * https://stackoverflow.com/questions/38414504/elasticsearch-read-and-write-consistency
    * https://www.alibabacloud.com/blog/elasticsearch-distributed-consistency-principles-analysis-3---data_594360
    * https://liqul.github.io/blog/things-about-replication-in-elasticsearch/
    * https://stackoverflow.com/questions/15694724/shards-and-replicas-in-elasticsearch
    * https://discuss.elastic.co/t/elastic-search-and-consistency/16825