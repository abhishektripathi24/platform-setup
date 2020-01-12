# Airflow [Cluster Setup] 
To know more about Airflow, visit https://github.com/apache/airflow

## Overview

From the official docs -

> Apache Airflow (or simply Airflow) is a platform to programmatically author, schedule, and monitor workflows.

> When workflows are defined as code, they become more maintainable, versionable, testable, and collaborative.

> Use Airflow to author workflows as directed acyclic graphs (DAGs) of tasks. The Airflow scheduler executes your tasks on an array of workers while following the specified dependencies. Rich command line utilities make performing complex surgeries on DAGs a snap. The rich user interface makes it easy to visualize pipelines running in production, monitor progress, and troubleshoot issues when needed.

## Setup

1. Prerequisites
    * Install Anaconda and gcc.
        ```bash
        wget https://repo.anaconda.com/archive/Anaconda3-5.3.1-Linux-x86_64.sh
        bash Anaconda3-5.3.1-Linux-x86_64.sh
        vim ~/.bashrc
           > export AIRFLOW_GPL_UNIDECODE=yes
           > export SLUGIFY_USES_TEXT_UNIDECODE=yes
           > export AIRFLOW_HOME=~/airflow
        source ~/.bashrc
        sudo yum install gcc
        ```
    
2. Create a virtual environment and install dependencies
    ```bash
    conda env create -f=env.yml (using following env.yml)
    conda activate airflow
    ``` 
    * env.yml
        ```yaml
         name: airflow
         channels:
          - conda-forge
          - anaconda
         dependencies:
          - python=3.6.7
          - cython=0.29.1
          - celery=4.2.1
          - flower=0.9.2
          - postgresql=10.5
          - gevent=1.3.6
          - greenlet=0.4.15
          - pip=18.1
          - pip:
            - psycopg2-binary==2.7.6.1
            - redis==2.10.6
            - apache-airflow==1.10.1
            - apache-airflow[async]==1.10.3
            - apache-airflow[celery]==1.10.3
            - apache-airflow[crypto]==1.10.3
            - apache-airflow[jdbc]==1.10.3
            - apache-airflow[hive]==1.10.3
            - apache-airflow[hdfs]==1.10.3
            - apache-airflow[ldap]==1.10.3
            - apache-airflow[mysql]==1.10.3
            -  apache-airflow[postgres]==1.10.3
            - apache-airflow[s3]==1.10.3
            - apache-airflow[redis]==1.10.3
        ```
3. Setup Airflow master node with Celery worker
    ```bash
    export AIRFLOW_HOME=~/airflow
    airflow initdb
    # Modify airflow.cfg using airflow.cfg (committed in this repo) and repeat the second step again.
    airflow webserver -p 8080
    airflow scheduler
    airflow flower
    airflow worker
    ```

4. Setup Workers
    * Follow the above steps to setup workers. (Worker nodes can even be created by cloning master node)
    * Start only the worker process on worker nodes. Tasks to be executed by them are communicated via Redis task queue.
        ```bash
        airflow worker
        ```
      
5. Synchronize data across master and worker nodes
    * This step ensures that whatever changes are made at the master nodes, corresponding incremental changes are replicated to worker nodes. This is needed because a job while executing at any worker node takes data from its local disk instead of master. If any change to dags is made at master nodes, all workers should be aware of it to incorporate those changes. This step can be avoided by keeping a common disk-volume for master and all workers.
    * Create ssh key-pair on master node
        ```bash 
        ssh-keygen -t rsa
        ```
    * Add the public key of the master to ~/.ssh/authorized_keys on the worker node
    * Put following cron in crontab at master node to sync data every minute across workers (Here we have just one external worker)
        ```bash
        crontab -e
        */1 * * * * cd $AIRFLOW_HOME && rsync -avz -e "ssh -i /home/ec2-user/.ssh/id_rsa" /home/ec2-user/airflow/ ec2-user@10.0.1.140:/home/ec2-user/airflow/ --exclude=logs/
        ``` 
        
6. Reverse Proxy (NGINX config)
    * Update following variable in airflow.cfg for airflow admin
        * base_url = http://my_host/myorg/airflow
    * Corresponding NGINX config
        ```buildoutcfg
        server {
          listen 80;
          server_name lab.mycompany.com;
        
          location /myorg/airflow/ {
              proxy_pass http://localhost:8080;
              proxy_set_header Host $host;
              proxy_redirect off;
              proxy_http_version 1.1;
              proxy_set_header Upgrade $http_upgrade;
              proxy_set_header Connection "upgrade";
          }
        }
        ```
    * Update following variable in airflow.cfg for flower dashboard
        * flower_url_prefix = /myorg/flower
    * Corresponding NGINX config
        ```buildoutcfg
            server {
                listen 80;
                server_name lab.mycompany.com;
            
                location /myorg/flower/ {
                    rewrite ^/myorg/flower/(.*)$ /$1 break;
                    proxy_pass http://localhost:5555;
                    proxy_set_header Host $host;
                    proxy_redirect off;
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection "upgrade";
                }
            }
        ```
    