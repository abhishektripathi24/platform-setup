# MySql [Master-Slave Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/mysql/images/mysql-logo.png" width="300" height="200"/>

To know more about mysql, visit https://dev.mysql.com/

## Overview

From the official docs -

> MySQL is an open-source database management system, commonly installed as part of the popular LAMP (Linux, Apache, MySQL, PHP/Python/Perl) stack. It uses a relational database and SQL (Structured Query Language) to manage its data.

## Setup
Installation of `Mysql 5.7` on `Ubuntu 18.04.3 LTS` - [ref](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/#apt-repo-fresh-install)

1. Common steps to be executed on both Master as well as Slave instance 
    * Update apt repository and install mysql-server
        ```bash
        sudo apt update
     
        # On Ubuntu 18.04, only the latest version of MySQL is included in the APT package repository by default. At the time of writing, that’s MySQL 5.7
        sudo apt install mysql-server -y
         
        # For fresh installations, run the included security script 
        sudo mysql_secure_installation
        
        # Verify installation
        sudo systemctl stop mysql.service
        sudo systemctl start mysql.service
        sudo systemctl enable mysql.service
        sudo systemctl status mysql.service
        ```
    * Enable root user's login by changing root user's auth plugin through mysql safe mode
        ```bash
        # First stop existing mysql process
        sudo systemctl stop mysql.service
        
        # Start msyqld in safe mode and update the auth plugin for root from `auth_socket` to `mysql_native_password`
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
    
        # Stop mysqld process and restart mysql service
        ps -ef | grep mysql
        sudo kill -9 <pid of mysqld_safe --skip-grant-tables> <pid of /usr/sbin/mysqld --basedir=/usr...>   
        sudo systemctl start mysql
        
        # Login to mysql shell from localhost
        mysql -uroot -proot 
        ```
    * Change data directory location to separate volume
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
        
        # Copy content of existing data directory
        sudo cp -r /var/lib/mysql/* /data/mysql/.
        sudo chown -R mysql:mysql /data/mysql    
    
        # Restart processes
        sudo systemctl restart apparmor
        sudo systemctl restart mysql
        ```
    * Update `/etc/mysql/mysql.conf.d/mysqld.cnf` to enable following
        ```bash
        # Connection settings
        bind-address = 0.0.0.0
 
        # Bin log settings
        server-id = 1 # If server-id = 1 for master instance, then it should be 2 for slave instance
        log_bin = mysql-bin.log
        log_bin_index = mysql-bin.log.index
        relay_log = mysql-relay-bin
        relay_log_index = mysql-relay-bin.index
        
        # GTID settings
        gtid_mode = ON
        enforce_gtid_consistency = ON
        ```
    *  Allow remote login for root user:
        ```bash
        # Login to mysql shell from server's localhost
        mysql -uroot -proot
        use mysql; 
        GRANT ALL ON *.* TO 'root'@'%' identified by 'root';
        FLUSH PRIVILEGES;
        ```
     * Apply and verify all changes
        ```bash
        # First restart mysql server
        sudo systemctl restart mysql
    
        # Check variables
        SHOW VARIABLES like '%log_bin%';
        SHOW VARIABLES like '%gtid%';
        ```

2. Steps to be executed on `Master`
    * Login to mysql shell and create a replication user and grant replication permission
        ```bash
        CREATE USER 'repl_user'@'<replica-instance-ip>' IDENTIFIED BY 'replica_password';
        GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'<replica-instance-ip>';
        FLUSH PRIVILEGES; 
        ```
    * Stop writes to master database
        ```bash
        FLUSH TABLES WITH READ LOCK;
        SET GLOBAL read_only = ON;
        ```
    * Check the master binlog position. This will be the starting point of replica
        ```bash
        SHOW MASTER STATUS\G
        
        # Sample output
        *************************** 1. row ***************************
                     File: mysql-bin.000327
                 Position: 1568347
             Binlog_Do_DB:
         Binlog_Ignore_DB:
        Executed_Gtid_Set: 98a2e938-5e0b-11ea-ad82-506b8d83ce00:1-102151
        1 row in set (0.00 sec)
        ```
    * Create master db dump 
        ```bash
        mysqldump -h <master-instance-ip> -u root -p --all-databases > master_dump.sql
        ```
    * Enable write on master
        ```bash
        SET GLOBAL read_only = OFF;
        UNLOCK TABLES;
        ```

3. Steps to be executed on `Slave`
    * Restore master dump in slave
        ```bash
        # Restore in replica
        mysql -uroot -p < master_dump.sql
        ``` 
    * Stop slave threads
        ```bash
        STOP SLAVE;
        ```
    * Point slave to master's checkpoint details
        ```bash
        CHANGE MASTER TO
        MASTER_HOST='<master-instance-ip>',
        MASTER_USER='repl_user',
        MASTER_PASSWORD='replica_password',
        MASTER_LOG_FILE='mysql-bin.000327', # This is the file name shown in SHOW MASTER STATUS\G
        MASTER_LOG_POS=1568347; # This is the position shown in SHOW MASTER STATUS\G
        ```
    * Allow slave to replicate from master by resuming slave threads
        ```bash
        START SLAVE;
        ```
    * Check replication status
        ```bash
        SHOW SLAVE STATUS\G
        
        # Sample output
        *************************** 1. row ***************************
                       Slave_IO_State: Waiting for master to send event
                          Master_Host: 10.11.16.30
                          Master_User: repl_user
                          Master_Port: 3306
                        Connect_Retry: 60
                      Master_Log_File: mysql-bin.000327
                  Read_Master_Log_Pos: 1878387
                       Relay_Log_File: mysql-relay-bin.000018
                        Relay_Log_Pos: 1878600
                Relay_Master_Log_File: mysql-bin.000327
                     Slave_IO_Running: Yes
                    Slave_SQL_Running: Yes
                      Replicate_Do_DB:
                  Replicate_Ignore_DB:
                   Replicate_Do_Table:
               Replicate_Ignore_Table:
              Replicate_Wild_Do_Table:
          Replicate_Wild_Ignore_Table:
                           Last_Errno: 0
                           Last_Error:
                         Skip_Counter: 0
                  Exec_Master_Log_Pos: 1878387
                      Relay_Log_Space: 1878894
                      Until_Condition: None
                       Until_Log_File:
                        Until_Log_Pos: 0
                   Master_SSL_Allowed: No
                   Master_SSL_CA_File:
                   Master_SSL_CA_Path:
                      Master_SSL_Cert:
                    Master_SSL_Cipher:
                       Master_SSL_Key:
                Seconds_Behind_Master: 0
        Master_SSL_Verify_Server_Cert: No
                        Last_IO_Errno: 0
                        Last_IO_Error:
                       Last_SQL_Errno: 0
                       Last_SQL_Error:
          Replicate_Ignore_Server_Ids:
                     Master_Server_Id: 1
                          Master_UUID: 98a2e938-5e0b-11ea-ad82-506b8d83ce00
                     Master_Info_File: /data/mysql-data/master.info
                            SQL_Delay: 0
                  SQL_Remaining_Delay: NULL
              Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
                   Master_Retry_Count: 86400
                          Master_Bind:
              Last_IO_Error_Timestamp:
             Last_SQL_Error_Timestamp:
                       Master_SSL_Crl:
                   Master_SSL_Crlpath:
                   Retrieved_Gtid_Set: 98a2e938-5e0b-11ea-ad82-506b8d83ce00:98628-102350
                    Executed_Gtid_Set: 98a2e938-5e0b-11ea-ad82-506b8d83ce00:1-98579:98628-102350,
        b52bf103-87c0-11ea-97b7-506b8d103aa4:1-8
                        Auto_Position: 0
                 Replicate_Rewrite_DB:
                         Channel_Name:
                   Master_TLS_Version:
  
        ```

4. Verify master-slave replication by changing something in master and reading it from replica.

5. Update `/etc/mysql/conf.d/mysql.cnf` to make slave `read-only`
    ```bash
    [mysqld]
    read_only
    
    NOTE: Even after this config - 2 type of users can still write to slave
           1. Users with `Super_priv` priviliges 
           2. User used by slave threads (`repl_user` as per this guide) 
    ``` 
    
6. Restart slave instance to apply read-only changes
    ```bash
    sudo systemctl restart mysql
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
        SET PASSWORD FOR 'jim'@'localhost' = PASSWORD('NewPass');
     
        # Update your own password
        SET PASSWORD = PASSWORD('NewPass');
    
        # Expire password for any user
        ALTER USER 'jim'@'localhost' PASSWORD EXPIRE;
          
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
        
        # E.g. Grants for debezium user
        GRANT SELECT, RELOAD, SHOW DATABASES, LOCK TABLES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'10.%';
        ``` 

2. Database dump and restore
    ```bash
    # Dump 1 db
    mysqldump -h <hostname> --port=3306 -u <username> -p <db-name> <optional:table-name> > <filename>.sql
    
    # Dump all dbs
    mysqldump -u <username> -p --all-databases > <filename>.sql
 
    # Restore 1 db
    mysql -uroot -proot db-name -f < file.sql
    
    # Restore all dbs
    mysql -uroot -p < file.sql
    ``` 

3. Database and table size
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
    log_bin_index = mysql-bin.log.index
    relay_log = mysql-relay-bin
    relay_log_index = mysql-relay-bin.index
    
    # GTID settings
    gtid_mode = ON
    enforce_gtid_consistency = ON
    ```

3. Grants dump and restore
    ```bash
    # Install package
    sudo apt install percona-toolkit 
 
    # Dump grants
    pt-show-grants --host <hostname> --user 'username' --password 'password' --ignore root@'%','mysql.session'@'localhost',rdsadmin@localhost,'rdsrepladmin'@'%' > grants.sql
    
    # Restore grants
    mysql -uroot -proot mysql < grants.sql 
    ```