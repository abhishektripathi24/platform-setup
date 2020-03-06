# MySql [Single Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/mysql/images/mysql-logo.png" width="300" height="200"/>

To know more about mysql, visit https://dev.mysql.com/

## Overview

From the official docs -

> MySQL is an open-source database management system, commonly installed as part of the popular LAMP (Linux, Apache, MySQL, PHP/Python/Perl) stack. It uses a relational database and SQL (Structured Query Language) to manage its data.

## Setup
Installation of `Mysql 5.7` on `Ubuntu 18.04.3 LTS` - [ref](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/#apt-repo-fresh-install)

1. Update apt repository and install mysql-server
    ```bash
    sudo apt update
 
    # On Ubuntu 18.04, only the latest version of MySQL is included in the APT package repository by default. At the time of writing, that’s MySQL 5.7
    sudo apt install mysql-server -y
     
    # For fresh installations, run the included security script 
    sudo mysql_secure_installation 
    ```

2. Verify installation
    ```bash
    sudo systemctl stop mysql.service
    sudo systemctl start mysql.service
    sudo systemctl enable mysql.service
    sudo systemctl status mysql.service
    ``` 

3. Enable root user's login by changing root user's auth plugin through mysql safe mode
    ```bash
    # first stop existing mysql process
    sudo systemctl stop mysql.service
    
    # start msyqld in safe mode and update the auth plugin for root from `auth_socket` to `mysql_native_password`
    sudo su 
    mkdir -p /var/run/mysqld
    chown mysql:mysql /var/run/mysqld
    sudo mysqld_safe --skip-grant-tables &
    mysql -uroot
    use mysql;
    SELECT user,authentication_string,plugin,host FROM mysql.user;
    UPDATE user set authentication_string=PASSWORD("root") WHERE user='root';
    UPDATE user set plugin="mysql_native_password" WHERE user='root'; 
    FLUSH PRIVILEGES;
    quit;

    # stop mysqld process and restart mysql service
    ps -ef | grep mysql
    sudo kill -9 <pid of mysqld_safe --skip-grant-tables> <pid of /usr/sbin/mysqld --basedir=/usr...>   
    sudo systemctl start mysql
    
    # login to mysql shell from localhost
    mysql -uroot -proot 
    ```

4. Update `/etc/mysql/mysql.conf.d/mysqld.cnf` to allow remote connections
    ```bash
    bind-address = 0.0.0.0
    ```

5.  Allow remote login for root user:
    ```bash
    # login to mysql shell from server's localhost
    mysql -uroot -proot
    use mysql; 
    GRANT ALL ON *.* TO 'root'@'%' identified by 'root';
    FLUSH PRIVILEGES;
    ```

6. Change data directory location to separate volume
    ```bash
    # Stop existing mysql server
    sudo systemctl stop mysql
 
    # Update path of new data dir
    sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
    > datadir = /data/mysql
 
    # Update apparmour to use new data dir location
    sudo vim /etc/apparmor.d/usr.sbin.mysqld
    ------------------------------       --------------------------
    |# Allow data dir access     |       |# Allow data dir access  |
    |  /var/lib/mysql/ r,        | --->  |  /data/mysql/ r,        |
    |  /var/lib/mysql/** rwk,    |       |  /data/mysql/** rwk,    | 
    ------------------------------       ---------------------------
    
    # Restart processes
    sudo systemctl restart apparmor
    sudo systemctl restart mysql
    ```

7.  Enable bin logs and gtid
    ```bash
    # Bin log settings
    server-id = 1
    log_bin = mysql-bin.log
    
    # GTID settings
    gtid_mode = ON
    enforce_gtid_consistency = ON
    
    # Verify
    SHOW VARIABLES like '%log_bin%';
    SHOW VARIABLES like '%gtid%';
    ```
    
## Administration
1. User Management - [[AWS](https://aws.amazon.com/premiumsupport/knowledge-center/duplicate-master-user-mysql/)]
    * User create/view/update/delete
        ```bash
        # Create user and allow login from localhost
        CREATE USER 'username'@'localhost' IDENTIFIED BY 'Abc123';
        CREATE USER 'username'@'10.%' IDENTIFIED BY 'Abc123';
    
        # View user info
        SELECT user, host, authentication_string FROM mysql.user WHERE user='root'; 
         
        # Update username
        RENAME USER 'username'@'localhost' TO 'newusername'@'localhost';
         
        # Delete user
        DROP USER 'username'@'localhost';
        ```
    * Password update/revoke
        ```bash
        # Update password for any user
        SET PASSWORD FOR jim@localhost = PASSWORD('NewPass');
     
        # Update your own password
        SET PASSWORD = PASSWORD('NewPass');
    
        # Expire password for any user
        ALTER USER jim@localhost PASSWORD EXPIRE;
          
        # List accounts without password 
        SELECT host, user FROM mysql.user WHERE authentication_string = '';
    
        # Identify users with duplicate passwords
        SELECT authentication_string, group_concat(user) FROM mysql.user GROUP BY authentication_string HAVING count(user)>1;             
        ```
    * Permission show/grant/revoke
        ```bash
        # View grants for user
        SHOW GRANTS FOR 'username';
        SHOW GRANTS FOR 'username'@'localhost';
  
        # Grant all permissions
        GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';
  
        # Grant specific permissions on specific database.table
        GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'username'@'%' WITH GRANT OPTION;
        E.g. GRANT SELECT ON database_name.* TO 'username'@'localhost';
             GRANT SELECT ON database_name.table_name TO 'username'@'localhost';
       
        # Revoke all permissions
        REVOKE ALL PRIVILEGES ON *.* FROM 'username'@'localhost';
  
        # Revoke specific permissions on specific database.table
        REVOKE ALL SELECT ON database_name.* FROM 'username'@'localhost';

        NOTE: Revoking a grant only possible if a grant was given earlier.
        ``` 

2. Database and table size
    ```bash
    SELECT 
       TABLE_NAME AS `Table`,
       ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)`
    FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = "database_name"
    ORDER BY (DATA_LENGTH + INDEX_LENGTH)
    DESC;
    ```

## Misc
1. Uninstall mysql:
    ```bash
    sudo apt -y purge mysql*
    sudo apt -y autoremove
    sudo rm -rf /etc/mysql
    sudo rm -rf /var/lib/mysql*
    ```

1. Sample custom `mysqld.cnf`: 
    ```bash
    #------------------------------------------------------------------------------
    # CUSTOMIZED OPTIONS
    #------------------------------------------------------------------------------
    
    # Add settings for extensions here
    
    # Connection settings
    bind-address = 0.0.0.0
    
    # Directory settings
    datadir = /data/mysql-data
    
    # Bin log settings
    server-id = 1
    log_bin = mysql-bin.log
    
    # GTID settings
    gtid_mode = ON
    enforce_gtid_consistency = ON
 
    ```
