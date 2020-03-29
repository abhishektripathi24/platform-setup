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
    pip install wheel
    source superset/bin/activate
    pip install superset==0.29.0rc7
    superset db upgrade 
    export FLASK_APP=superset
    flask fab create-admin
    superset load_examples
    superset init
    superset run -h 0.0.0.0 -p 8080 --with-threads --reload --debugger    
    ```

3. Create `superset_config.py` to set following - 
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

4. Set `PYTHONPATH` to allow superset to read `superset_config.py` for all superset commands
    ```bash
    export PYTHONPATH=/opt/superset
    ``` 
    NOTE: Re-run commands with prefix `superset` as in step 2, to reconfigure superset installation as per `superset_config.py`.

5. Install Database dependencies for adding datasource in superset
    ```bash
    # PostgreSQL
    pip install psycopg2-binary
   
    # MySQL
    sudo yum install mysql-devel # (CentOS)
    sudo apt-get install libmysqlclient-dev # (Ubuntu)
    sudo pip install mysqlclient pymysql
    ``` 

6. Run webserver via gunicorn
    ```bash
    # Install Gevent
    pip install gevent
   
    # Start the gunicorn process
    gunicorn \
          -w 10 \
          -k gevent \
          --timeout 600 \
          -b  0.0.0.0:8090 \
          --limit-request-line 0 \
          --limit-request-field_size 0 \
          --statsd-host localhost:8125 \
          --error-logfile /opt/superset/logs/error_logs \
          --access-logfile /opt/superset/logs/access_logs \
          --log-level debug \
          superset:app
       
    # Note: Avoid gevent coroutine if there is high mem-util.
    ```

7. Setup another server with exactly same configuration and configure a load balancer in-front of these two nodes for HA. 

8. If your linux distro supports systemd, you can supervise superset process under it. The corresponding systemd service file is present in this repo at [this](systemd) location.

## References
* https://github.com/instacart/superset/blob/master/docs/installation.rst

## Errors
* Run `pip install werkzeug==0.16.0` for below error:
     ```bash
     (superset) root@hostname:/opt/superset# superset db upgrade 
     File "/opt/superset/lib/python3.6/site-packages/flask_appbuilder/filemanager.py", line 9, in <module>
        from werkzeug import secure_filename
     ImportError: cannot import name 'secure_filename'
    ```
* Run `pip uninstall pandas` then `pip install pandas==0.23.4` for below error:
    ```bash
    (superset) root@hostname:/opt/superset# superset db upgrade
    File "/opt/superset/lib/python3.6/site-packages/superset/dataframe.py", line 14, in <module>
        from pandas.core.common import _maybe_box_datetimelike
    ImportError: cannot import name '_maybe_box_datetimelike'
    ```
* Run `pip uninstall sqlalchemy` then `pip install sqlalchemy==1.2.18` for below error:
    ```bash
    (superset) root@hostname:/opt/superset# superset db upgrade
    File "/opt/superset/lib/python3.6/site-packages/sqlalchemy/orm/query.py", line 2632, in _join_determine_implicit_left_side
        "Can't determine which FROM clause to join "
    sqlalchemy.exc.InvalidRequestError: Can't determine which FROM clause to join from, there are multiple FROMS which can join to this entity. Please use the .select_from() method to establish an explicit left side, as well as providing an explcit ON clause if not present already to help resolve the ambiguity.
    ```
* Run `fabmanager create-admin --app superset` for below errors:
    ```bash
    (superset) root@hostname:/opt/superset# flask fab create-admin
    Usage: flask [OPTIONS] COMMAND [ARGS]...
    
    Error: No such command "fab".
    ```