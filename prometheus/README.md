# Prometheus [Single Node Setup - Prometheus, BlackboxExporter, AlertManager]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/prometheus/images/prometheus-logo.png" width="650" height="200"/>

To know more about Prometheus, https://prometheus.io/

## Overview
From the official docs of prometheus, blackbox_exporter and alertmanager -

> Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.

> Blackbox Exporter probes endpoints over HTTP, HTTPS, DNS, TCP or ICMP protocols, returning detailed metrics about the request, including whether or not it was successful and how long it took to receive a response.

> The Alertmanager handles alerts sent by client applications such as the Prometheus server. It takes care of deduplicating, grouping, and routing them to the correct receiver integration such as email, PagerDuty, or OpsGenie. It also takes care of silencing and inhibition of alerts.

## Setup Prometheus
Installation of `Prometheus 2.16` on `Ubuntu 18.04.3 LTS` - [ref](https://github.com/prometheus/prometheus)

1. Download and extract standalone linux binary - [ref](https://prometheus.io/docs/prometheus/latest/getting_started/#downloading-and-running-prometheus)
    ```bash
    cd /opt
    wget https://github.com/prometheus/prometheus/releases/download/v2.16.0/prometheus-2.16.0.linux-amd64.tar.gz
    tar -xzf prometheus-2.16.0.linux-amd64.tar.gz
    mv prometheus-2.16.0.linux-amd64 prometheus
    cd prometheus
    ```

2. Update `prometheus.yml` for scrape config under section `scrape_config` - [ref](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
    ```bash
    scrape_configs:
      # scrape targets through static configs
      - job_name: 'static-scrape-job'
        metrics_path: /metrics
        scrape_interval: 10s
        static_configs:
           - targets: ["ip1:9273","ip2:9273"]
    
      # scrape targets through dynamic discovery using ec2 service-discovery mechanism
       - job_name: 'ec2-scrape-job'
         metrics_path: '/metrics'
         scrape_interval: 15s
         scrape_timeout: 10s
         ec2_sd_configs:
           - region: ap-southeast-1
             access_key: YOURACCESSKEY
             secret_key: YOURSECRETKEY
             port: 9273
             refresh_interval:  300s
         # relabel_configs allow advanced modifications to any target and its labels before scraping 
         relabel_configs:
           - source_labels: ['__meta_ec2_tag_Name']
             target_label: instance_name
           - source_labels: ['__meta_ec2_availability_zone']
             target_label: availability_zone
           - source_labels: [__meta_ec2_instance_id]
             target_label: instance_id
           - source_labels: [__meta_ec2_instance_type]
             target_label: instance_type
           - source_labels: [__meta_ec2_private_ip]
             target_label: private_ip
           - source_labels: ['__meta_ec2_public_ip']
             target_label: public_ip
           - source_labels: ['__meta_ec2_subnet_id']
             target_label: subnet_id
           - source_labels: ['__meta_ec2_vpc_id']
             target_label: vpc_id
           - source_labels: ['__meta_ec2_tag_Environment']
             target_label: environment
           - source_labels: ['__meta_ec2_tag_Business_Vertical']
             target_label: business_vertical
           - source_labels: ['__meta_ec2_tag_Cluster']
             target_label: cluster_name
           - source_labels: ['__meta_ec2_tag_Project']
             target_label: project
           # filter out those that do not contain a Monitor tag
           - source_labels: ['__meta_ec2_tag_Monitor_Application']
             action: keep
             regex: yes
           - source_labels: ['__meta_ec2_tag_Business_Vertical']
             action: keep
             regex: blueleaf
           - source_labels: ['__meta_ec2_tag_Environment']
             action: keep
             regex: production
    ``` 

3. Start the process
    ```bash
    ./prometheus --config.file=prometheus.yml --storage.tsdb.path=/data/prometheus
    ``` 

4. If you linux distro supports systemd, you can supervise prometheus process under it. The corresponding systemd service file is present in this repo at [this](systemd) location. 

## Setup Blackbox Exporter
Installation of `Bloackbox Exporter 0.16.0` on `Ubuntu 18.04.3 LTS` - [ref](https://github.com/prometheus/blackbox_exporter)

1. Download and extract standalone linux binary
    ```bash
    cd /opt
    wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.16.0/blackbox_exporter-0.16.0.linux-amd64.tar.gz
    tar -xzf blackbox_exporter-0.16.0.linux-amd64.tar.gz
    mv blackbox_exporter-0.16.0.linux-amd64 blackbox_exporter
    cd blackbox_exporter
    ```

2. Just start the blackbox_exporter process
    ```bash
    ./blackbox_exporter
    ``` 

3. The list of endpoints to be probed is located in the Prometheus configuration file as part of the Blackbox Exporter’s targets directive. Update `prometheus.yml` for scrape config under section `scrape_config`. No need to change anything in blackbox exporter config - [ref](https://github.com/prometheus/blackbox_exporter)
    ```bash
    scrape_configs:
     - job_name: 'website-scrape-job'
        metrics_path: /probe # Blackbox Exporter runs on port 9115 with metrics available on the /probe endpoint.
        scrape_interval: 60s
        scrape_timeout: 5s
        params:
          module: [http_2xx]  # Look for a HTTP 200 response.
        static_configs:
          - targets:
            - https://www.google.com  # Target to probe with https.
        relabel_configs:
          - source_labels: [__address__]
            target_label: __param_target
          - source_labels: [__param_target]
            target_label: site
          - target_label: __address__
            replacement: 127.0.0.1:9115 # Blackbox Exporter runs on port 9115 with metrics available on the /probe endpoint.
    ```

4. If you linux distro supports systemd, you can supervise blackbox_exporter process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

5. Visiting below url will return metrics for a HTTP probe against google.com. The `probe_success` metric indicates if the probe succeeded. Adding a `debug=true` parameter will return debug information for that probe.
    ```bash
    http://localhost:9115/probe?target=google.com&module=http_2xx&debug=true
    ```
    
## Setup Alertmanager
Installation of `Alertmanager 0.20.0` on `Ubuntu 18.04.3 LTS` - [ref](https://github.com/prometheus/alertmanager)

1. Download and extract standalone linux binary
    ```bash
    cd /opt
    wget https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-amd64.tar.gz
    tar -xzf alertmanager-0.20.0.linux-amd64.tar.gz
    mv alertmanager-0.20.0.linux-amd64 alertmanager
    cd alertmanager
    ```
    
2. Update `alertmanager.yml` - [ref1](https://github.com/prometheus/alertmanager), [ref2](https://prometheus.io/docs/alerting/configuration/)
    ```bash
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'email-smtp.eu-west-1.amazonaws.com:587'
      smtp_auth_username: 'AKIA5GD****'
      smtp_auth_password: 'BNq3PaOFDS8****'
      smtp_from: 'alerts@blueleaf.com'
      smtp_require_tls: true
    
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
       
      # Default receiver
      receiver: 'team-X-mails'
       
      # The child route trees
      routes:
      - match:
          business_vertical: 'abc'
        receiver: 'team-Y-mails'
        continue: true # Continue comparison
      - match:
          cluster_name: 'redis-prod'
        receiver: 'team-Z-mails'
        continue: true 
      - match_re: # regex for label - job (regex weren't working when last checked)
          job: 'job-infra-.*'
        receiver: 'team-Z-mails'
      - match_re: 
          service: ^(foo1|foo2|baz)$
        receiver: 'team-Y-mails'
    
    inhibit_rules:
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname', 'instance_name']
    
    receivers:
    - name: 'team-X-mails'
      email_configs:
      - to: 'rec1@blueleaf.com, rec2@blueleaf.com'
        send_resolved: true # Send alert resolution mail 
    - name: 'team-Y-mails'
      email_configs:
      - to: 'rec1@blueleaf.com, rec2@blueleaf.com'
        send_resolved: true 
    - name: 'team-Z-mails'
      email_configs:
      - to: 'rec1@blueleaf.com, rec2@blueleaf.com'
        send_resolved: true 
    ```

3. Update `prometheus.yml` to connect alertmanager with prometheus 
    ```bash
    # Alertmanager configuration
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
           - localhost:9093
    
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
        - "rules/*.yaml"
      # - "first_rules.yml"
      # - "second_rules.yml"
    ```

4. Put rules under `prometheus/rules`. Samples available [here](rules). - [ref](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)

5. Start alertmanager process
    ```bash
    ./alertmanager
    ```

6. If you linux distro supports systemd, you can supervise alertmanager process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## References
* https://blog.ruanbekker.com/blog/2019/05/17/prometheus-series-of-tutorials-for-your-guide-to-epic-metrics/
* https://ops.tips/blog/simulating-aws-tags-in-local-prometheus/
* https://kbild.ch/blog/2019-02-18-awsprometheus/
* https://prometheus.io/docs/prometheus/latest/configuration/configuration/
