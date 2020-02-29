# Let's Encrypt [Free SSL/TLS certificate]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/letsencrypt/images/letsencrypt-logo.png" width="600" height="190"/>

To know more about Let's Encrypt, visit https://letsencrypt.org/

## Overview

From the official docs of let's encrypt -

> A nonprofit Certificate Authority providing TLS certificates to million websites. To enable HTTPS on your website, you need to get a certificate (a type of file) from a Certificate Authority (CA). Let’s Encrypt is a CA. In order to get a certificate for your website’s domain from Let’s Encrypt, you have to demonstrate control over the domain. With Let’s Encrypt, you do this using software that uses the ACME protocol which typically runs on your web host.

## Installation using Certbot ACME client 

Steps to get a `wildcard certificate` for NGINX-Ubuntu 18.04.3 LTS combo from letsencrypt. For a FQDN certificate refer the steps under default section in [certbot instructions](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx).

1. Add Certbot PPA 
    ```bash
    sudo apt-get update
    sudo apt-get install software-properties-common
    sudo add-apt-repository universe
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    ```

2. Install Certbot
    ```bash
    sudo apt-get install certbot python-certbot-nginx
    ```

3. Install DNS plugin
    ```bash
    sudo apt-get install python3-certbot-dns-<PLUGIN>
 
    # E.g. if DNS provider is AWS Route53
    sudo apt-get install python3-certbot-dns-route53
    ```

4. Setup credentials as per DNS provider - [ref](https://certbot.eff.org/docs/using.html#dns-plugins)
    ```bash
    # E.g. if DNS provider is AWS Route53, put credentials at ~/.aws/credentials
    [default]
    aws_access_key_id=KODIA5GD****
    aws_secret_access_key=yE2zSELPaCDHkN***
    ```

5. Get a wildcard certificate for your domain
    ```bash
    sudo certbot certonly --dns-route53 --preferred-challenges=dns --email abhishek.tripathi2421@gmail.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.stg.blueleaf.com --manual-public-ip-logging-ok 
    ```
    Note: use `--dry-run` flag for testing.
    
    Sample output:
    ```bash
    ubuntu@ip-10-8-69-215:~$ sudo certbot certonly --dns-route53 --preferred-challenges=dns --email abhishek.tripathi2421@gmail.com --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.stg.blueleaf.com --manual-public-ip-logging-ok
    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Found credentials in shared credentials file: ~/.aws/credentials
    Plugins selected: Authenticator dns-route53, Installer None
    
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    Would you be willing to share your email address with the Electronic Frontier
    Foundation, a founding partner of the Let's Encrypt project and the non-profit
    organization that develops Certbot? We'd like to send you email about our work
    encrypting the web, EFF news, campaigns, and ways to support digital freedom.
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    (Y)es/(N)o: n
    Obtaining a new certificate
    Performing the following challenges:
    dns-01 challenge for stg.blueleaf.com
    Waiting 10 seconds for DNS changes to propagate
    Waiting for verification...
    Cleaning up challenges
    
    IMPORTANT NOTES:
     - Congratulations! Your certificate and chain have been saved at:
       /etc/letsencrypt/live/stg.blueleaf.com/fullchain.pem
       Your key file has been saved at:
       /etc/letsencrypt/live/stg.blueleaf.com/privkey.pem
       Your cert will expire on 2020-05-29. To obtain a new or tweaked
       version of this certificate in the future, simply run certbot
       again. To non-interactively renew *all* of your certificates, run
       "certbot renew"
     - Your account credentials have been saved in your Certbot
       configuration directory at /etc/letsencrypt. You should make a
       secure backup of this folder now. This configuration directory will
       also contain certificates and private keys obtained by Certbot so
       making regular backups of this folder is ideal.
     - If you like Certbot, please consider supporting our work by:
    
       Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
       Donating to EFF:                    https://eff.org/donate-le
    
    ```

6. Automatic renewal
    ```bash
    sudo certbot renew
    ``` 
    Note: use `--dry-run` flag for testing.
        
    Sample output:
    ```bash
    ubuntu@ip-10-8-69-215:~$ sudo certbot renew --dry-run
    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    Processing /etc/letsencrypt/renewal/stg.blueleaf.com.conf
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    Cert not due for renewal, but simulating renewal for dry run
    Found credentials in shared credentials file: ~/.aws/credentials
    Plugins selected: Authenticator dns-route53, Installer None
    Renewing an existing certificate
    Performing the following challenges:
    dns-01 challenge for stg.blueleaf.com
    Waiting 10 seconds for DNS changes to propagate
    Waiting for verification...
    Cleaning up challenges
    
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    new certificate deployed without reload, fullchain is
    /etc/letsencrypt/live/stg.blueleaf.com/fullchain.pem
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    ** DRY RUN: simulating 'certbot renew' close to cert expiry
    **          (The test certificates below have not been saved.)
    
    Congratulations, all renewals succeeded. The following certs have been renewed:
      /etc/letsencrypt/live/stg.blueleaf.com/fullchain.pem (success)
    ** DRY RUN: simulating 'certbot renew' close to cert expiry
    **          (The test certificates above have not been saved.)
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    ```

## References
* https://certbot.eff.org/docs/using.html
* https://loganmarchione.com/2018/09/lets-encrypt-wildcard-certificates-with-certbot-on-nginx/
* https://medium.com/@bmatthewshea/small-thing-97c23338f272
