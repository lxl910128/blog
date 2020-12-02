---
title: docker配置中从mysql
categories:
- mysql, docker
tags:
- mysql
keywords:
- docker, mysql, 主从数据库, 数据同步
---
# 概述
配置数据库主从，可以实现从库数据自动与主库同步，进而实现读写分离。mysql实现主从数据同步的核心思路是从数据库获取主数据bin-log，根据bin-log将主数据库执行过的操作再执行一遍。本文将介绍如何在docker中搭建mysql主从测试环境。

# 步骤

1. 正确安装docker、docker-compse

2. 找一个目录(本文使用/opt/mysql)创建`docker-compose.yml`文件以及master和slave文件夹。master、slave分别存放2个mysql的配置、数据、日志

3. ```yaml
   #编辑docker-compose.yml文件如下
   version: '3.5'
   
   x-logging: &default-logging
     options:
       max-size: "10m"
       max-file: "10"
   services:
     master:
       # 使用mariadb 10.2版本
       image: mariadb:10.2
       container_name: mysql
       restart: always
       ports:
         - 3306:3306
       volumes:
         # 数据目录映射
         - /opt/mysql/master/data:/var/lib/mysql
         # 日志目录映射
         - /opt/mysql/master/log:/var/log/mysql
         # dump文件映射（不重要）
         - /opt/mysql/master/dump:/app/dump
         # 配置文件映射
         - /opt/mysql/master/my.cnf:/etc/mysql/my.cnf
       environment:
         TZ: "Asia/Shanghai"
         # mysql 密码
         MYSQL_ROOT_PASSWORD: root
       logging: *default-logging
       command:
         [
           "--character-set-server=utf8mb4",
           "--collation-server=utf8mb4_unicode_ci",
           "--max_connections=1024",
         ]
   
   
     mysql-slave:
       image: mariadb:10.2
       container_name: mysql
       restart: always
       ports:
       # slave 使用3307端口
         - 3307:3307
       volumes:
         - /opt/mysql/slave/data:/var/lib/mysql
         - /opt/mysql/slave/log:/var/log/mysql
         - /opt/mysql/slave/dump:/app/dump
         - /opt/mysql/slave/my.cnf:/etc/mysql/my.cnf
       environment:
         TZ: "Asia/Shanghai"
         MYSQL_ROOT_PASSWORD: root
       logging: *default-logging
       command:
         [
           "--character-set-server=utf8mb4",
           "--collation-server=utf8mb4_unicode_ci",
           "--max_connections=1024",
         ]
   ```

4. 使用`docker pull mariadb:10.2`拉取mariadb 10.2版本，此版本相当于mysql 5.7

5. 在master、slave中创建`data`、`log`、`dump`目录以及`my.cnf`文件

6. ```shell
   # master 和 slave 的默认 my.cnf 如下
   # MariaDB database server configuration file.
   #
   # You can copy this file to one of:
   # - "/etc/mysql/my.cnf" to set global options,
   # - "~/.my.cnf" to set user-specific options.
   #
   # One can use all long options that the program supports.
   # Run program with --help to get a list of available options and with
   # --print-defaults to see which it would actually understand and use.
   #
   # For explanations see
   # http://dev.mysql.com/doc/mysql/en/server-system-variables.html
   
   # This will be passed to all mysql clients
   # It has been reported that passwords should be enclosed with ticks/quotes
   # escpecially if they contain "#" chars...
   # Remember to edit /etc/mysql/debian.cnf when changing the socket location.
   [client]
   port		= 3306
   socket		= /var/run/mysqld/mysqld.sock
   
   # Here is entries for some specific programs
   # The following values assume you have at least 32M ram
   
   # This was formally known as [safe_mysqld]. Both versions are currently parsed.
   [mysqld_safe]
   socket		= /var/run/mysqld/mysqld.sock
   nice		= 0
   
   [mysqld]
   #
   # * Basic Settings
   #
   #user		= mysql
   pid-file	= /var/run/mysqld/mysqld.pid
   socket		= /var/run/mysqld/mysqld.sock
   port		= 3306
   basedir		= /usr
   datadir		= /var/lib/mysql
   tmpdir		= /tmp
   lc_messages_dir	= /usr/share/mysql
   lc_messages	= en_US
   skip-external-locking
   #
   # Instead of skip-networking the default is now to listen only on
   # localhost which is more compatible and is not less secure.
   #bind-address		= 127.0.0.1
   #
   # * Fine Tuning
   #
   max_connections		= 100
   connect_timeout		= 5
   wait_timeout		= 600
   max_allowed_packet	= 16M
   thread_cache_size       = 128
   sort_buffer_size	= 4M
   bulk_insert_buffer_size	= 16M
   tmp_table_size		= 32M
   max_heap_table_size	= 32M
   #
   # * MyISAM
   #
   # This replaces the startup script and checks MyISAM tables if needed
   # the first time they are touched. On error, make copy and try a repair.
   myisam_recover_options = BACKUP
   key_buffer_size		= 128M
   #open-files-limit	= 2000
   table_open_cache	= 400
   myisam_sort_buffer_size	= 512M
   concurrent_insert	= 2
   read_buffer_size	= 2M
   read_rnd_buffer_size	= 1M
   #
   # * Query Cache Configuration
   #
   # Cache only tiny result sets, so we can fit more in the query cache.
   query_cache_limit		= 128K
   query_cache_size		= 64M
   # for more write intensive setups, set to DEMAND or OFF
   #query_cache_type		= DEMAND
   #
   # * Logging and Replication
   #
   # Both location gets rotated by the cronjob.
   # Be aware that this log type is a performance killer.
   # As of 5.1 you can enable the log at runtime!
   #
   # Error logging goes to syslog due to /etc/mysql/conf.d/mysqld_safe_syslog.cnf.
   #
   # we do want to know about network errors and such
   #log_warnings		= 2
   #
   # Enable the slow query log to see queries with especially long duration
   #slow_query_log[={0|1}]
   slow_query_log_file	= /var/log/mysql/mariadb-slow.log
   long_query_time = 10
   #log_slow_rate_limit	= 1000
   #log_slow_verbosity	= query_plan
   
   #log-queries-not-using-indexes
   #log_slow_admin_statements
   #
   # The following can be used as easy to replay backup logs or for replication.
   # note: if you are setting up a replication slave, see README.Debian about
   #       other settings you may need to change.
   #server-id		= 1
   #report_host		= master1
   #auto_increment_increment = 2
   #auto_increment_offset	= 1
   #log_bin			= /var/log/mysql/mariadb-bin
   #log_bin_index		= /var/log/mysql/mariadb-bin.index
   # not fab for performance, but safer
   #sync_binlog		= 1
   expire_logs_days	= 10
   max_binlog_size         = 100M
   # slaves
   #relay_log		= /var/log/mysql/relay-bin
   #relay_log_index	= /var/log/mysql/relay-bin.index
   #relay_log_info_file	= /var/log/mysql/relay-bin.info
   #log_slave_updates
   #read_only
   #
   # If applications support it, this stricter sql_mode prevents some
   # mistakes like inserting invalid dates etc.
   #sql_mode		= NO_ENGINE_SUBSTITUTION,TRADITIONAL
   #
   # * InnoDB
   #
   # InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
   # Read the manual for more InnoDB related options. There are many!
   default_storage_engine	= InnoDB
   innodb_buffer_pool_size	= 256M
   innodb_log_buffer_size	= 8M
   innodb_file_per_table	= 1
   innodb_open_files	= 400
   innodb_io_capacity	= 400
   innodb_flush_method	= O_DIRECT
   #
   # * Security Features
   #
   # Read the manual, too, if you want chroot!
   # chroot = /var/lib/mysql/
   #
   # For generating SSL certificates I recommend the OpenSSL GUI "tinyca".
   #
   # ssl-ca=/etc/mysql/cacert.pem
   # ssl-cert=/etc/mysql/server-cert.pem
   # ssl-key=/etc/mysql/server-key.pem
   
   #
   # * Galera-related settings
   #
   [galera]
   # Mandatory settings
   #wsrep_on=ON
   #wsrep_provider=
   #wsrep_cluster_address=
   #binlog_format=row
   #default_storage_engine=InnoDB
   #innodb_autoinc_lock_mode=2
   #
   # Allow server to accept connections on all interfaces.
   #
   #bind-address=0.0.0.0
   #
   # Optional setting
   #wsrep_slave_threads=1
   #innodb_flush_log_at_trx_commit=0
   
   [mysqldump]
   quick
   quote-names
   max_allowed_packet	= 16M
   
   [mysql]
   #no-auto-rehash	# faster start of mysql but no tab completion
   
   [isamchk]
   key_buffer		= 16M
   
   #
   # * IMPORTANT: Additional settings that can override those from this file!
   #   The files must end with '.cnf', otherwise they'll be ignored.
   #
   !includedir /etc/mysql/conf.d/
   ```

7. 修改master的配置文件my.cnf，在[mysqld]中增加以下内容以开启bin-log：

   1. log_bin                   = /var/lib/mysql/mariadb_bin.log				#开启 Binlog 并写明存放日志的位置
   2. log_bin_index             = /var/lib/mysql/mariadb-bin.index      # 指定索引文件的位置
   3. expire_logs_days          = 7                                                            # 超过7天的binlog将被删除
   4. server_id                 = 0001                                                            # 指定一个serverID
   5. binlog_format             = ROW                                                       # 设置bin-log的日志模式

8. 修改slave的配置文件my.cnf：

   1. [client]中的port = 3306 改为port=3307
   2. [mysqld]中的port=3306改为port=3307
   3. [mysqld]中增加server_id=0002

9. `docker exec -it mysql bash`进入master容器，`mysql -u root -p`进入mysql

10. 创建用于让从数据库同步主数据库数据的用户`repl`

    ```mysql
    MariaDB [(none)]> create user 'repl'@'%' identified by 'repl';
    MariaDB [(none)]> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';
    MariaDB [(none)]> flush privileges;
    ```

11. 查看目前bin-log状态，记住File名字，和Position偏移位置

    ```mysql
    MariaDB [(none)]> show master status;
    +--------------------+----------+--------------+------------------+
    | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
    +--------------------+----------+--------------+------------------+
    | mariadb_bin.000003 |      807 |              |                  |
    +--------------------+----------+--------------+------------------+
    1 row in set (0.00 sec)
    ```

    

12. 另启动一个shell进入slave的mysql客户端

13. 执行如下命令，开始跟踪主库bin-log，注意设置`MASTER_LOG_FILE`、`MASTER_LOG_POS`

    ```mysql
    MariaDB [(none)]> CHANGE MASTER TO MASTER_HOST='192.168.55.7', MASTER_PORT=3306,  MASTER_USER='repl', MASTER_PASSWORD='repl', MASTER_LOG_FILE='mariadb_bin.000003', MASTER_LOG_POS=807;
    MariaDB [(none)]> start slave;
    ```

14. 在slave使用`show slave status\G`查看是否同步开启成功，如果成功`Slave_IO_Running`和`Slave_SQL_Running`会显示为`YES`

    ```mysql
    
    MariaDB [(none)]> show slave status \G
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 192.168.55.7
                      Master_User: repl
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mariadb_bin.000003
              Read_Master_Log_Pos: 1672
                   Relay_Log_File: mysqld-relay-bin.000002
                    Relay_Log_Pos: 1282
            Relay_Master_Log_File: mariadb_bin.000003
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
              Exec_Master_Log_Pos: 1672
                  Relay_Log_Space: 1592
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
                   Master_SSL_Crl:
               Master_SSL_Crlpath:
                       Using_Gtid: No
                      Gtid_IO_Pos:
          Replicate_Do_Domain_Ids:
      Replicate_Ignore_Domain_Ids:
                    Parallel_Mode: conservative
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
    1 row in set (0.00 sec)
    ```

15. 现在在主库进行操作从库就会同步了

# 注意点

1. 设置slave是指定了从bin-log的哪个offset开始跟踪，如果有创建数据库操作在此offset之前，那么从库将不会自动创建该库。
2. 如果跟踪offset滞后，导致部分数据从库没有跟踪，如果主库操作了这些数据，则会造成从库跟踪失败，导致同步停止。针对此情况有2个方案：
   1. 取消从库对某个数据库的跟踪
   2. 执行命令跳过出错的语句`stop slave;SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1; START SLAVE;`
3. 配置从库时务必注意offset合理设置，保证开启slave的状态和主库offset指向的状态一致。

# 主从原理

https://www.cnblogs.com/fengzheng/p/13401783.html

