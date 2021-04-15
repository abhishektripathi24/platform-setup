# Elastic Stack [EK Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/elastic-stack/images/elastic-logo.png" width="750" height="250"/>

To know more about Elastic Stack, visit https://www.elastic.co/elastic-stack

## Overview
From the official docs -

> Elasticsearch is the distributed search and analytics engine at the heart of the Elastic Stack.

> Logstash and Beats facilitate collecting, aggregating, and enriching your data and storing it in Elasticsearch.

> Kibana enables you to interactively explore, visualize, and share insights into your data and manage and monitor the stack. Elasticsearch is where the indexing, search, and analysis magic happen.

## Log-processing using Elasticsearch's ingest pipeline
1. Setup elasticsearch as described [here](../elasticsearch).
2. Create an ingest pipeline with name `sample_ingest_pipeline`. Some sample ingest pipelines with respective log formats are available [here.](ingest-pipelines.txt)

## Log-ingestion using Filebeat
1. Download and extract the latest version - [ref](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation.html) 
    ```bash
    curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.5.2-linux-x86_64.tar.gz
    tar xzvf filebeat-7.5.2-linux-x86_64.tar.gz
    cd filebeat-7.5.2-linux-x86_64
    ``` 
   
 2. Configure `filebeat.yml` to read log file and output to elasticsearch's ingest pipeline
    ```yaml
    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/application.log
      multiline.pattern: '^\['
      multiline.negate: true
      multiline.match: after
    
    setup.template.enabled: false

    output.elasticsearch:
      hosts: ["localhost:9200"]
      pipeline: sample_ingest_pipeline
      index: "custom-index-name-%{+yyyy.MM.dd}"
    ``` 
    
3. Start filebeat process
    ```bash
    ./filebeat
    ```
   
4. (Optional) Sample ebextensions for installing and starting filebeat via AWS elasticbeanstalk are present [here](ebextensions).
 
 > Note: There is some problem in custom index renaming with Filebeat-7.5.2, even after following the section "change the index name" in the [official documentation](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-template.html). However custom index works fine in Filebeat-6.8.2 as per the above mentioned config .  
 
## Log-visualization using Kibana
1. Download and extract the latest version - [ref](https://www.elastic.co/downloads/kibana) 
    ```bash
    cd /opt
    wget https://artifacts.elastic.co/downloads/kibana/kibana-7.5.2-linux-x86_64.tar.gz   
    tar xzvf kibana-7.5.2-linux-x86_64.tar.gz
    mv kibana-7.5.2-linux-x86_64 kibana
    cd kibana
    ``` 
   
2. Update `config/kibana.yml` - ([x-pack ref](https://www.elastic.co/guide/en/kibana/7.5/settings-xpack-kb.html))
    ```yaml
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://localhost:9200"]
    logging.dest: /var/log/kibana/kibana.log
 
    # Security settings (optional)
    console.enabled: false
    map.includeElasticMapsService: false
    uiSettings.overrides.defaultRoute: /app/kibana#/dashboards
    xpack.security.enabled: false
    xpack.apm.enabled: false
    xpack.grokdebugger.enabled: true
    xpack.searchprofiler.enabled: false
    xpack.graph.enabled: false
    xpack.ml.enabled: false
    xpack.reporting.enabled: false
    ```
   
3. Start kibana process
    ```bash
    ./bin/kibana
    ```

 ## References
 * https://www.elastic.co/guide/en/beats/filebeat/current/index.html
 * https://www.elastic.co/guide/en/kibana/current/index.html