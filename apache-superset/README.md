# Superset [Multi-Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/apache-superset/images/superset-logo.png" width="700" height="200"/>

To know more about Superset, visit https://github.com/instacart/superset

## Overview
From the official docs -

> Apache Superset (incubating) is a modern, enterprise-ready business intelligence web application.

## Setup
Installation of `Apache Superset 0.29.0rc7` on `Ubuntu 18.04.3 LTS` - [ref](https://superset.incubator.apache.org/installation.html)

1. Install dependencies
    ```bash
    sudo apt-get install build-essential libssl-dev libffi-dev python3.6-dev python-pip libsasl2-dev libldap2-dev
    sudo apt-get install python3-venv
    ```

2. Create a virtual env `superset` using python3.6 and install superset
    ```bash
    cd /opt
    python3.6 -m venv superset
    source superset/bin/activate
    pip install apache-superset
    superset db upgrade
    export FLASK_APP=superset
    flask fab create-admin
    superset load_examples
    superset init
    superset run -h 0.0.0.0 -p 8080 --with-threads --reload --debugger    
    ```

3. Create superset_config.py to set following - (Run superset related command again after creating superset_config.py as ran in step 2)
    * Superset database backend.
    * LDAP config for SSO.
    * Query timeouts.
    * Domain sharding.
    
    With `vim /opt/superset/superset_config.py`, the final config looks like this -
    ```bash
    #---------------------------------------------------------
    # Superset specific config
    #---------------------------------------------------------
    ROW_LIMIT = 5000
    DEBUG = False
    SUPERSET_WEBSERVER_PORT = 8080
    #---------------------------------------------------------
    
    #---------------------------------------------------------
    # Flask App Builder configuration
    #---------------------------------------------------------
    # Your App secret key
    SECRET_KEY = '\2\1thisismyscretkey\1\2\e\y\y\h'
    
    # The SQLAlchemy connection string to your database backend
    # This connection defines the path to the database that stores your
    # superset metadata (slices, connections, tables, dashboards, ...).
    # Note that the connection information to connect to the datasources
    # you want to explore are managed directly in the web UI
    # SQLALCHEMY_DATABASE_URI = 'sqlite:////path/to/superset.db'
    SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://username:password@<ip/hostname>:5432/superset'
    
    # Flask-WTF flag for CSRF
    WTF_CSRF_ENABLED = True
    # Add endpoints that need to be exempt from CSRF protection
    WTF_CSRF_EXEMPT_LIST = []
    # A CSRF token that expires in 1 year
    WTF_CSRF_TIME_LIMIT = 60 * 60 * 24 * 365
    
    # Set this API key to enable Mapbox visualizations
    MAPBOX_API_KEY = ''
    
    # assuming org-name: blueleaf
    from flask_appbuilder.security.manager import AUTH_LDAP
    AUTH_TYPE = AUTH_LDAP
    AUTH_LDAP_SERVER = "ldap://<ip/hostname>:389"
    AUTH_LDAP_USE_TLS = False
    AUTH_LDAP_SEARCH = "dc=blueleaf,dc=com"
    AUTH_LDAP_BIND_USER = "cn=superset@blueleaf.com,ou=applications,ou=users,dc=blueleaf,dc=com"
    AUTH_LDAP_BIND_PASSWORD = "password"
    AUTH_LDAP_UID_FIELD = "cn"
    AUTH_LDAP_EMAIL_FIELD = "cn"
    
    # Query timeout config
    SQLLAB_TIMEOUT = 900
    SUPERSET_WEBSERVER_TIMEOUT = 900
    
    # Domain sharding config
    SUPERSET_WEBSERVER_DOMAINS = "comma separated superset server public-ip/domains for sharding queries from UI"
    ```

4. Install Database dependencies for adding datasource in superset
    ```bash
    # PostgreSQL
    pip install psycopg2-binary
    # MySQL
    sudo yum install mysql-devel # (CentOS)
    sudo apt-get install libmysqlclient-dev # (Ubuntu)
    sudo pip install mysqlclient
    ``` 

5. Run webserver via gunicorn
    ```bash
    gunicorn \
          -w 10 \
          -k gevent \
          --timeout 600 \
          -b  0.0.0.0:8080 \
          --limit-request-line 0 \
          --limit-request-field_size 0 \
          --statsd-host localhost:8125 \
          --error-logfile /opt/superset/logs/error_logs \
          --access-logfile /opt/superset/logs/access_logs \
          --log-level debug \
          superset:app
       
    # Note: Avoid gevent coroutine if there is high mem-util.
    ```

6. Setup another server with exactly same configuration and configure a load balancer in-front of these two nodes for HA. 

7. If you linux distro supports systemd, you can supervise superset process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## References
* https://github.com/instacart/superset/blob/master/docs/installation.rst