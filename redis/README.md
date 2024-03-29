# Redis [Sentinel Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/redis/images/redis-logo.png" width="600" height="200"/>

To know more about Redis Sentinel, visit https://redis.io/topics/sentinel

## Overview
![](images/sentinel-combined.png)

From the official docs -

> Redis Sentinel provides high availability for Redis. In practical terms this means that using Sentinel you can create a Redis deployment that resists without human intervention certain kinds of failures.

## Setup
Installation of `Redis 5.0.7` on `Ubuntu 18.04.3 LTS` - [ref](https://github.com/antirez/redis)

1. Download and compile from the source code
    ```bash
    sudo apt update
    sudo apt install gcc
    sudo apt install make
    sudo apt-get install -y tcl
    wget http://download.redis.io/releases/redis-5.0.7.tar.gz
    tar xzf redis-5.0.7.tar.gz
    cd redis-5.0.7
    make distclean
    make
    make test
    ```

2. Configure OS: This will remove all the warnings in redis-server logs
    ```bash
    # Swap space
    sudo sysctl vm.swappiness=1 # Temporary
    echo vm.swappiness=1 >> /etc/sysctl.conf # Permanent
    
    # Overcommit memory
    sudo sysctl vm.overcommit_memory=1 # Temporary
    echo vm.overcommit_memory=1 >> /etc/sysctl.conf # Permanent
 
    # somaxconn
    echo 1024 > /proc/sys/net/core/somaxconn # Temporary
    echo net.core.somaxconn=1024 >> /etc/sysctl.conf # Permanent

    # THP (disable-thp.service is available in redis/systemd in this repo or refer - https://www.stephenrlang.com/2018/01/disabling-transparent-huge-pages-in-linux/)
    echo never > /sys/kernel/mm/transparent_hugepage/enabled # Temporary
    sudo systemctl enable disable-thp.service && sudo systemctl start disable-thp.service # Permanent
    ```

3. Configure Redis-Server: update `redis.conf`
    * Configure Master: 
        ```properties
        # Comment the line if you want to allow clients to connect redis from all network interfaces and not restricted to loopback interface
        bind 127.0.0.1
        
        # Disable auth if you don't want authentication.
        protected-mode no

        # Set supervised systemd if you want to manage redis via systemd. The systemd files for redis-server and redis-sentinel can be found under redis/systemd in this repo.
        supervised systemd

        # Set log file & location
        logfile "/var/log/redis/redis-server.log"

        # Allow accept writes even if BGSAVE failed
        stop-writes-on-bgsave-error no

        # Allow saving data files (`dump.rdb`, `appendonly.aof`) at a volume different than root
        dir "/data"

        # Allow append only file persistence
        appendonly yes

        # Define when to trigger automatic rewrite of the append only file
        auto-aof-rewrite-percentage 20
        auto-aof-rewrite-min-size 64mb
        
        # Disable critical commands. (Please add these after testing the setup. Otherwise you won't be able to issue these commands
        rename-command FLUSHDB ""
        rename-command FLUSHALL ""
        rename-command DEBUG ""
        
        # Enabled active defragmentation (WARNING: THIS FEATURE IS EXPERIMENTAL BUT LOAD TESTED IN PROD)
        activedefrag yes 
        
        # Limit max memory usage (e.g. 15Gbs) [Caution!! use it wisely]
        maxmemory 16106127360 
        ```
    * Configure Slave:
        ```
        # Repeat and add all the properties of master.
        ...
  
        # Set initial master 
        replicaof  <masterip> <masterport>
        ```

4. Configure Redis-Sentinel: update `sentinel.conf`
    * Set following for a 3 node sentinel setup on each sentinel node -
        ```properties
        # Set log file & location      
        logfile "/var/log/redis/redis-sentinel.log"
        
        # Set quorum and failover policy
        sentinel monitor mymaster <ip-address-of-initial-master-redis-server> 6379 2
        sentinel down-after-milliseconds mymaster 5000
        sentinel parallel-syncs mymaster 1
        sentinel failover-timeout mymaster 10000
        ```
        > Note: Please do not copy sentinel.conf to other servers if you have started sentinel process even once. 
        
        > Reason: Upon starting the sentinel process, it writes a unique id for itself in the sentinel.conf (sentinel myid 1e05acbfea43ecc0ff0c976ede504805a6342db3). And if you replicate (copy-paste) this sentinel.conf to other sentinel servers, all of them will behave as if you have a single sentinel node instead of 3 nodes in the cluster, as all of them will have the same id.
        
        > Moral of the story: Each sentinel node should have unique id in the sentinel cluster and this id is generated automatically by sentinel process itself.

5. Start the process on each server
    ```bash
    ./src/redis-server redis.conf
    ./src/redis-sentinel sentinel.conf
    ```
   
6. Testing the cluster setup
    * Verify redis conf changes are properly loaded. Following example shows maxmemory - 
        ```bash
        cd <path/to/redis-5.0.7>/src
        ./redis-cli -p 6379
        127.0.0.1:6379> config get maxmemory
        1) "maxmemory"
        2) "16106127360"
        ```
    * Verify that sentinels see each other. If, for ex, you are setting up 3 sentinels, the property `num-other-sentinels` should be `2` upon starting all the sentinel nodes. Run following commands on any sentinel node - 
        ```bash
        cd <path/to/redis-5.0.7>/src
        ./redis-cli -p 26379
        127.0.0.1:26379> sentinel master mymaster
        33) "num-other-sentinels"
        34) "2"
        35) "quorum"
        36) "2"
        ```
    * Simulate failover-scenario -
        * Run following command on redis-master. And verify the logs of sentinel and redis-slave as they vote and switch the master.
            ```bash
            cd <path/to/redis-5.0.7>/src
            ./redis-cli -p 6379 
            127.0.0.1:6379> DEBUG sleep 30
            ```

## Misc
1. Backup data from AWS ElastiCache
    * Create backup of your ElastiCache.
    * Migrate backup to S3 using aws cli -
        * Create an S3 bucket.
        * Provide permission to the bucket to receive data from elastic cache -
            * Go to Permission -> Access Control List
            * Under the section "Access for other AWS accounts" click on "Add Account"
            * Add following id to it - 540804c33a284a299d2547575ce1010f2312ef3da9b3a053c8bc45bf233e4353 (this is canonical id for singapore region. For other regions, refer [Exporting a ElastiCache Backup](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/backups-exporting.html))
            * Finally provide all the read-write permission to this account and click save.
        * Run following command from terminal - 
            ```bash
            aws elasticache copy-snapshot \
                --source-snapshot-name <backup-name> \
                --target-snapshot-name my-exported-backup \
                --target-bucket <s3-bucket-name>
            ```

## Administration
List of all commands is available here - [ref](https://redis.io/commands)
1. Force trigger rdb - `BGSAVE`
2. Force trigger AOF rewrite - `BGREWRITEAOF`
3. Find biggest objects in redis - `./redis-cli --bigkeys`
4. Memory management cmds - `MEMORY DOCTOR, MEMORY STATS`
5. Generate memory reports using `rdbtools` - [ref](https://github.com/sripathikrishnan/redis-rdb-tools#generate-memory-report) 

## Monitoring  
* Monitoring redis using metrics exposed by [Telegraf](https://docs.influxdata.com/telegraf/v1.13/):
    * Installation `https://docs.influxdata.com/telegraf/v1.13/introduction/installation/`
    * Configuration `https://github.com/influxdata/telegraf/tree/master/plugins/inputs/redis`
    * A sample telegraf configuration and corresponding grafana dashboard is available [here](monitoring).
    * Dashboard constituents version -
        * Grafana - `Grafana v5.3.4 (69630b9)`
        * Telegraf - `Telegraf 1.13.1 (git: HEAD 0c175724)`

## References
* [What redis deployment do you need?](https://blog.octo.com/what-redis-deployment-do-you-need/) 
* [A medium blog may be?](https://medium.com/@amila922/redis-sentinel-high-availability-everything-you-need-to-know-from-dev-to-prod-complete-guide-deb198e70ea6)
* [Performance tuning - Datadog](https://www.datadoghq.com/pdf/Understanding-the-Top-5-Redis-Performance-Metrics.pdf)
* [OOM Killer](https://www.percona.com/blog/2019/08/02/out-of-memory-killer-or-savior/)
* [Consistency for Sentinel setup](https://www.ctolib.com/docs/sfile/redis-doc-cn/cn/topics/sentinel.html)