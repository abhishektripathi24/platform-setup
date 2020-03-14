# NGINX [Single Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/nginx/images/nginx-logo.png" width="550" height="200"/>

To know more about NGINX, visit https://www.nginx.com/

## Overview
From the official docs -

> NGINX is open source software for web serving, reverse proxying, caching, load balancing, media streaming, and more. It started out as a web server designed for maximum performance and stability. In addition to its HTTP server capabilities, NGINX can also function as a proxy server for email (IMAP, POP3, and SMTP) and a reverse proxy and load balancer for HTTP, TCP, and UDP servers.

## Setup
Installation of `NGINX 1.14.0` on `Ubuntu 18.04.3 LTS` - [ref](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)

1. Install NGINX
    ```bash
    sudo apt update
    sudo apt install nginx 
    ```

2. Verify installation and enable (autorun the process) upon os reboots
    ```bash
    sudo systemctl stop nginx
    sudo systemctl start nginx
    sudo systemctl status nginx
    sudo systemctl enable nginx
    ``` 

3. Create config files
    ```bash
    # create a file under sites-available <eg. custom-proxy>
    sudo touch /etc/nginx/sites-available/custom-proxy
      
    # create symlink of this file under sites-enabled (nginx reads sites-enabled for configs)
    ln -s ../sites-available/custom-proxy custom-proxy
   ```

## Configure NGINX as Reverse Proxy
1. Configure settings for *reverse proxy* by writing following content into `sites-available/custom-proxy` - [ref](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) 
    ```bash
    # Redirect all http requests on the host application.blueleaf.com to https
    server {
      listen 80;
      server_name application.blueleaf.com;
      return 301 https://$host$request_uri;
    }
   
    # Serve all https requests on the host application.blueleaf.com
    server {
      listen 443 ssl;
      server_name application.blueleaf.com;
      ssl_certificate "/etc/ssl/certs/certs-chain.crt";
      ssl_certificate_key "/etc/ssl/private/cert.key";
    
      location / {
        proxy_pass http://127.0.0.1:8080;
      }
    
      # This will redirect application.blueleaf.com/somepath -> 127.0.0.1:8080/somepath
      # Or application.blueleaf.com/somepath/abc -> 127.0.0.1:8080/somepath//abc (Hence the error due to //) 
      location /somepath/ {
        rewrite ^/somepath(.*) /$1 break; # note the '/' before $1
        proxy_pass http://127.0.0.1:8080;
      }
    
      # This will redirect application.blueleaf.com/someotherpath/abc -> 127.0.0.1:8080/somepath/abc
      location /someotherpath/ {
        rewrite ^/someotherpath(.*) $1 break; # note there is no '/' before $1
        proxy_pass http://<some-other-app-server-ip>:8083;
      }
    }
    ```

2. Reload live nginx process with the new config
    ```bash
   sudo systemctl reload nginx 
   ``` 

3. Examples
    * Host based routing `(assume 3 hosts: example1.com, example2.com, example3.com)`
        ```bash
        server {
           listen       80;
           server_name  example1.com;
       
           location / {
               proxy_pass http://127.0.0.1:8080;
           }
        }
       
       server {
           listen       80;
           server_name  example2.com;
       
           location / {
               proxy_pass http://127.0.0.1:8181;
           }
       }
       
       server {
           listen       80;
           server_name  example3.com;
       
           location / {
               proxy_pass http://127.0.0.1:8282;
           }
       }
        ```
   
    * Path based routing `(assume 3 paths: /, /blog, /mail)`
        ```bash
        server {
           listen       ...;
           ...
           location / {
               proxy_pass http://127.0.0.1:8080;
           }
           
           location /blog {
               proxy_pass http://127.0.0.1:8181;
           }
       
           location /mail {
               proxy_pass http://127.0.0.1:8282;
           }
           ...
        }
        ```
   
   * Path rewriting `(to avoid the path/sub-path being passed on to the proxied server)` - [rules](https://www.nginx.com/blog/creating-nginx-rewrite-rules/)
       ```bash
       server {
           listen       ...;
           ...
           location / {
               proxy_pass http://127.0.0.1:8080;
           }
           
           location /blog {
               rewrite ^/blog(.*) /$1 break;
               proxy_pass http://127.0.0.1:8181;
           }
       
           location /mail {
               rewrite ^/mail(.*) /$1 break;
               proxy_pass http://127.0.0.1:8282;
           }
           ...
       }
       ```

## Configure NGINX as Load Balancer
* https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
* http://nginx.org/en/docs/http/load_balancing.html
* http://nginx.org/en/docs/http/ngx_http_upstream_module.html

## References
* https://gist.github.com/soheilhy/8b94347ff8336d971ad0
* https://linuxize.com/post/how-to-install-nginx-on-ubuntu-18-04/