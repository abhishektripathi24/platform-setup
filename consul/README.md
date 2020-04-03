# Consul [Cluster Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/consul/images/consul-logo.png" width="450" height="180"/>

To know more about Consul, visit https://www.consul.io/

## Overview
From the official docs -

> Consul is a service networking solution to connect and secure services across any runtime platform and public or private cloud.

## Consul Setup
Installation of `Consul v1.7.2` on `Ubuntu 18.04.3 LTS` - [ref](https://learn.hashicorp.com/consul)

1. Download and extract standalone linux binary - [ref](https://www.consul.io/downloads.html)
    ```bash
    wget https://releases.hashicorp.com/consul/1.7.2/consul_1.7.2_linux_amd64.zip
    unzip consul_1.7.2_linux_amd64.zip
    sudo mv consul /usr/local/bin/
    which consul
    rm consul_1.7.2_linux_amd64.zip
    ```

2. Create a consul secret and provide it to `encrypt` param in consul server config of all the nodes
    ```bash
    consul keygen
    ```

3. Create config file for all the nodes at `/etc/consul.d/config.json` with the following properties (the following properties are for a cluster of 3 nodes) - [ref](https://www.consul.io/docs/agent/options.html)
    ```bash
    {
        "bootstrap_expect": 3,
        "client_addr": "0.0.0.0",
        "datacenter": "dc-name",
        "data_dir": "/data/consul",
        "domain": "consul",
        "enable_script_checks": true,
        "dns_config": {
            "enable_truncate": true,
            "only_passing": true
        },
        "enable_syslog": true,
        "encrypt": "QXzRb+qG/Fno7+EaCiCLEWgGWW90crbSpv+cMD4y60s=", # Copied from step 2
        "leave_on_terminate": true,
        "log_file": "/var/log/consul/consul.log",
        "log_level": "INFO",
        "node_name": "consul-node-1", # Don't forget to change the node name for each node 
        "rejoin_after_leave": true,
        "server": true,
        "start_join": [
            "<node-1-ip-or-hostname>",
            "<node-2-ip-or-hostname>",
            "<node-3-ip-or-hostname>"
        ],
        "ui": true
    }
    ```

4. Start the process on each server
    ```bash
    sudo consul agent -bind=<ip-of-that-machine> -config-dir /etc/consul.d/
    ``` 

5. Test the setup
    ```bash
    # via ui
    http://localhost:8500/ui
    
    # via console 
    consul members
    
    Node           Address           Status  Type    Build  Protocol  DC       Segment
    consul-node-1  10.11.18.58:8301  alive   server  1.7.2  2         nutanix  <all>
    consul-node-2  10.11.18.59:8301  alive   server  1.7.2  2         nutanix  <all>
    consul-node-3  10.11.18.60:8301  alive   server  1.7.2  2         nutanix  <all>
    ```

6. If you linux distro supports systemd, you can supervise consul-server process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## Usage
1. Register service via HTTP API - [ref](https://www.consul.io/api/agent/service.html#register-service)
    ```http request
    curl -X PUT \
      http://<consul-node-hostname-or-ip>:8500/v1/agent/service/register \
      -H 'Content-Type: application/json' \
      -H 'Postman-Token: 245711cc-9b6a-4042-b1e4-b9f06b8f3bc7' \
      -H 'cache-control: no-cache' \
      -d '{
    	"id": "<service-id>",
        "name":"<service-name>",
        "tags":["ticket-id"],
        "address": "<machine-ip>",
        "port": <service-port>,
        "checks": [
          {
            "http": "http://<machine-ip>:<service-port>/actuator/health",
            "interval": "10s"
          }
        ]
    }'
    ```
    
2. Deregister service via HTTP API - [ref](https://www.consul.io/api/agent/service.html#deregister-service)
    ```http request
    curl -X PUT \
      http://<consul-node-hostname-or-ip>:8500/v1/agent/service/deregister/<service-id> \
      -H 'Postman-Token: 91f8550e-8a8f-46c7-b55d-e2efef23f229' \
      -H 'cache-control: no-cache'
    ```

3. Query service via DNS interface - [ref](https://learn.hashicorp.com/consul/getting-started/services)
    ```bash
    dig @<any-consul-cluster-node-ip> -p 8600 <service-name>.service.consul
    e.g. dig +short @10.11.18.60 -p 8600 oauth-service-stg.service.consul
    ```

<br/>

## Setup DNS Forwarding  
Forward DNS requests for domain `.consul` on port `53` to `8600` via `Unbound 1.6.7` on `Ubuntu 18.04.3 LTS` - [ref](https://learn.hashicorp.com/consul/security-networking/forwarding) 

1. Install `Unbound 1.6.7` on all the nodes where consul was installed
    ```bash
    sudo apt update
    sudo apt install unbound -y
    ```
 
2. Create config file for forwarding `.consul` domain request from port `53` to `8600` at `/etc/unbound/unbound.conf.d/consul.conf` with the following properties - [ref](https://www.consul.io/docs/agent/options.html)
    ```bash
    #Allow insecure queries to local resolvers
    server:
      do-not-query-localhost: no
      domain-insecure: "consul"
    
    #Add consul as a stub-zone
    stub-zone:
      name: "service.consul."
      stub-addr: 127.0.0.1@8600
    
    #Forward domains other than .consul to the following nameservers
    forward-zone:
      name: "."
      forward-addr: <your-name-server-ip-1>@53
      forward-addr: <your-name-server-ip-2>@53
    ```
    
3. Stop `systemd-resolved.service` on all the nodes where unbound is running
    ```bash
    sudo systemctl stop systemd-resolved.service
    sudo systemctl disable systemd-resolved.service
    ```

4. Update `/etc/resolv.conf` to replace `nameserver 127.0.0.53` to following
    ```bash
    nameserver 127.0.0.1
    ```
    
5. If you face following msg everytime a command is run via sudo - `sudo: unable to resolve host <hostname>`, update `/etc/hosts` as follows - [ref](https://askubuntu.com/questions/59458/error-message-sudo-unable-to-resolve-host-none)
    ```bash
    127.0.1.1       <hostname>
    ```
 
6. Test the setup - [ref](https://www.consul.io/docs/agent/dns.html)
    ```bash
    dig +short @<unbound/consul-server-ip> -p 53 <service-name>.service.consul
    ```

7. Update nameservers in `/etc/systemd/resolved.conf` on the nodes which want to resolve `.service.consul` domain
    ```bash
    [Resolve]
    DNS=<unbound-ip-1> <unbound-ip-2> <unbound-ip-3>

    E.g.
    [Resolve]
    DNS=10.11.18.58 10.11.18.59 10.11.18.60 
    ``` 

8. Restart `systemd-resolved.service`
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart systemd-resolved.service
    ```

9. Verify name resolution - [ref](https://www.consul.io/docs/agent/dns.html)
    ```bash
    dig +short <service-name>.service.consul
    ```

 <br/>
 
## Consul Template and NGINX Load Balancing Setup
Installation of `Consul Template 0.24.1` on `Ubuntu 18.04.3 LTS` - [ref1](https://learn.hashicorp.com/consul/integrations/nginx-consul-template), [ref2](https://github.com/hashicorp/consul-template)

1. Download and extract standalone linux binary - [ref](https://releases.hashicorp.com/consul-template/)
    ```bash
    wget https://releases.hashicorp.com/consul-template/0.24.1/consul-template_0.24.1_linux_386.zip
    unzip consul-template_0.24.1_linux_386.zip
    sudo cp consul-template /usr/local/bin/
   ```
   
2. Reference config from [hashicorp github](https://github.com/hashicorp/consul-template#configuration-file-format) and update for change. Sample working config is available [here](consul-template/consul-template-config.hcl) and templates [here](consul-template/consul-nginx-template.ctmpl) - [ref](https://github.com/hashicorp/consul-template/blob/master/examples/nginx.md)
    ```json
    # provide consul address
    address = "<consul-node-1-ip-or-hostname>:8500"
   
    # provide source template location
    source = "/usr/local/etc/consul-nginx-template.ctmpl"
     
    # provide destination template location
    destination = "/etc/nginx/conf.d/consul-nginx.conf"
    
    NOTE: Additionally you need to disable auth and other things to make the config work. Please go to the reference to view a working config.
    ``` 

3. Start the process 
    ```bash
   consul-template -config consul-template-config.hcl
   ```

4. Test the setup by visiting `/etc/nginx/conf.d/consul-nginx.conf` and checking the `server_name` in the generated `server` blocks
    ```bash
    curl --header 'Host: <value of server_name>' 'http://<nginx-ip>:80/any/api/path/of/service'
    ``` 

5. If your linux distro supports systemd, you can supervise consul-template process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

# References
* https://danielparker.me/nginx/consul-template/consul/nginx-consul-template/