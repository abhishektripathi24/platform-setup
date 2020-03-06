# Grafana [Single Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/grafana/images/grafana-logo.png" width="400" height="100"/>

To know more about Grafana, visit https://grafana.com/

## Overview
From the official docs of grafana -

> Grafana is the open source analytics and monitoring solution for every database.

## Setup
Installation of `Grafana 6.6.2` on `Ubuntu 18.04.3 LTS` - [ref](https://grafana.com/docs/grafana/latest/installation/debian/)

1. Download and extract standalone linux binary
    ```bash
    cd /opt
    wget https://dl.grafana.com/oss/release/grafana-6.6.2.linux-amd64.tar.gz
    tar -zxvf grafana-6.6.2.linux-amd64.tar.gz
    mv grafana-6.6.2.linux-amd64 grafana
    cd grafana
    ```

2. Update configs for user authentication and alert notification - [ref](https://grafana.com/docs/grafana/latest/installation/configuration/)
    * As grafana defaults are stored in `defaults.ini` and it doesn't allow config overriding, create a copy of sample configuration `conf/sample.ini`.
        ```bash
        sudo cp conf/sample.ini conf/custom.ini
        ```
    * Server configuration - 
        ```bash
        [server]
        # The public facing domain name used to access grafana from a browser
        domain = monitoring.blueleaf.com
  
        # The full public facing url you use in browser, used for redirects and emails
        # If you use reverse proxy and sub path specify full url (with sub path)
        root_url = %(protocol)s://%(domain)s:%(http_port)s/ # if reverse proxy is not used
        # or 
        root_url =  %(protocol)s://%(domain)s/grafana/ # if reverse proxy is used and nginx uses /grafana as the path

        # Log web requests
        router_logging = true
        ```
    
    * Ldap configuration - [ref](https://grafana.com/docs/grafana/latest/auth/ldap/)
        * Update `conf/custom.ini`
            ```bash
            [auth.ldap]
            enabled = true
            config_file = /opt/grafana/conf/ldap.toml
            allow_sign_up = true
            ```
        * Update `conf/ldap.toml`
            ```bash
             # assuming org-name: blueleaf
             [[servers]]
             host = "ldap://<ip/hostname>"
             port = 389
             use_ssl = false
             start_tls = false
             ssl_skip_verify = false
             bind_dn = "cn=admin,dc=blueleaf,dc=com"
             bind_password = 'admin'
             search_filter = "(cn=%s)"
             search_base_dns = ["dc=blueleaf,dc=com"]
            ```
    *  SMTP configuration - [ref](https://grafana.com/docs/grafana/latest/alerting/notifications/)
        * Update `conf/custom.ini`
            ```bash
            [smtp]
            enabled = true
            host = email-smtp.eu-west-1.amazonaws.com:587
            user = AKIA5CRGDQUDHCXYNYPF62CR
            password = BNq3PaOFDS8YGeqCRPuINSDIIEeuiRLu48sf1yewUbiqARCR
            from_address = alerts@blueleaf.com
            from_name = Grafana
            ```

3. Start the process
    ```bash
    ./bin/grafana-server
    ``` 
    
4. If you linux distro supports systemd, you can supervise grafana-server process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## References
* https://grafana.com/docs/grafana/latest/alerting/rules/
