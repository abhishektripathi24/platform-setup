# MongoDB [Single Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/mongo/images/mongodb-logo.png" width="450" height="130"/>

To know more about MongoDB, visit https://www.mongodb.com/

## Overview
From the official docs -

> MongoDB is a general purpose, document-based, distributed database built for modern application developers and for the cloud era.
  
## Setup
Installation of `MongoDB v3.6.3` on `Ubuntu 18.04.3 LTS` - [ref](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/)

1. Install mongodb  
    ```bash
    sudo apt update
    sudo apt install -y mongodb
    ```
   
2. Verify installation
    ```bash
    mongo --eval 'db.runCommand({ connectionStatus: 1 })'        
    ```
