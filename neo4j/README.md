# Neo4j [With ineo - Neo4j instance manager]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/neo4j/images/neo4j-logo.png" width="500" height="250"/>

To know more about Neo4j, visit https://neo4j.com/

## Overview

From the official docs -

> A graph database is a database designed to treat the relationships between data as equally important to the data itself. It is intended to hold data without constricting it to a pre-defined model. Instead, the data is stored like we first draw it out – showing how each individual entity connects with or is related to others.
  
## Setup

Installation on ubuntu 18.04.3 LTS

1. Install Java8
    ```bash
    sudo apt update
    sudo apt install openjdk-8-jdk openjdk-8-jre
    java -version
    ```

2. Install ineo - [ref](https://github.com/cohesivestack/ineo)
    ```bash
    curl -sSL http://getineo.cohesivestack.com | bash -s install -d ~/ineo-custom-dir-path
    source ~/.bashrc
    ```
3. Orchestration -
    ```bash
    # Download a specific version of neo4j
    ineo create -v 3.2.0 -p7474 <instance_name> 
 
    # allow remote access
    vim ~/ineo-custom-dir-path/instances/<intance_name>/conf/neo4j.conf
    > dbms.shell.host=0.0.0.0
    ineo restart <instance_name>  
 
    # Helper commands
    ineo instances/status/start/stop/restart/destroy/delete-db
    ```