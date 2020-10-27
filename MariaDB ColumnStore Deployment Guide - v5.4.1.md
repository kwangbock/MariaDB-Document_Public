# MariaDB ColumnStore Deployment Guide - 5.4.1

MariaDB ColumnStore 5.4.1 Deployment Guide



## ※ Version History

| Version | Date          | Description   |
| ------- | ------------- | ------------- |
| 1.0     | 26, OCT, 2020 | Initial Draft |
|         |               |               |



**Table of Contents**

[TOC]

## 1) Deploy Architecture

+ 3 Node Deployment on RHEL7.7
+ Utilize **<u>AWS S3</u>** for ColumnStore Storage



## 2) Prerequisites / System Preparation

Systems hosting ColumnStore Instances require some additional configuration prior to installation.



### Firewalls and SELinux

| **TCP Ports** | **Description**                              |
| :------------ | :------------------------------------------- |
| 3306          | Port used for MariaDB Client traffic         |
| 8600-8630     | Port range used for inter-node communication |
| 8640          | Port used by CMAPI                           |
| 8700          | Port used for inter-node communication       |
| 8800          | Port used for inter-node communication       |

MariaDB ColumnStore requires port **3306**, **8600 to 8650**, **8700**, and **8800**. On some operating systems, this may require configuring the firewall and SELinux to permit TCP traffic over these ports.

```bash
$ sudo getenforce
$ sestatus

# Disable SELinux Temporarily
$ sudo setenforce 0

# Disable SELinux Permanently
$ sudo vi /etc/selinux/config
SELINUX=disabled
```



### Character Encoding

When using MariaDB ColumnStore, it is recommended to set the system's locale to UTF-8.

```bash 
$ sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```

On RHEL8 and CentOS8 install the following and set the system's locale

```bash
$ sudo yum install glibc-locale-source glibc-langpack-en
$ sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```



### Dependent Packages

There are several dependant packages. Please install `epel-release` package first, it will automatically include any necessry packages during `YUM` installation. If your servers can't connect `YUM` repository, you need to install all the dependant packages manually.

```bash
$ sudo yum -y install epel-release

$ sudo yum -y install wget boost expect perl perl-DBI openssl zlib file sudo libaio rsync snappy net-tools numactl-libs nmap jemalloc rsyslog htop git gcc jq mlocate nano unzip python-devel python36-requests python-requests python3-pip python2-pip

$ sudo pip install boto
```



### Optimizing Kernel Parameters

MariaDB ColumnStore performs best when certain Linux kernel parameters are optimized.

Set the relevant kernel parameters in a sysctl configuration file. For proper change management, it is recommended to set them in a ColumnStore-specific configuration file.

Create a `/etc/sysctl.d/90-mariadb-columnstore.conf` file. 

```bash
# Increase the TCP max buffer size
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Increase the TCP buffer limits
# min, default, and max number of bytes to use
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# don't cache ssthresh from previous connection
net.ipv4.tcp_no_metrics_save = 1

# for 1 GigE, increase this to 2500
# for 10 GigE, increase this to 30000
net.core.netdev_max_backlog = 2500

# optimize Linux to cache directories and inodes
vm.vfs_cache_pressure = 10

# minimize swapping
vm.swappiness = 10
```

The set the same kernel parameters at runtime using the `sysctl` command:

```bash
$ sudo sysctl --load=/etc/sysctl.d/90-mariadb-columnstore.conf
```



## 3) Installation

### Install via Package Tarball 

Retrieve your Customer Download Token at *https://customers.mariadb.com/downloads/token/* and substitute for `customer_download_token` in the following directions.



#### Download Package Tarball

```bash
# Download ES 10.5
$ wget https://dlm.mariadb.com/<customer-token>/mariadb-enterprise-server/10.5.6-4/rpm/rhel/mariadb-enterprise-10.5.6-4-centos-7-x86_64-rpms.tar

# CMAPI 1.1
$ wget https://dlm.mariadb.com/<customer-token>/mariadb-enterprise-server/10.5.6-4/cmapi/mariadb-columnstore-cmapi-1.1.tar.gz

# Python `boto` Package
$ wget https://files.pythonhosted.org/packages/23/10/c0b78c27298029e4454a472a1919bde20cb182dab1662cec7f2ca1dcc523/boto-2.49.0-py2.py3-none-any.whl
```



#### Install ES 10.5 and ColumnStore

```bash
$ rpm -ivh MariaDB-common-10.5.6_4-1.el7.x86_64.rpm MariaDB-compat-10.5.6_4-1.el7.x86_64.rpm
$ yum install MariaDB-client-10.5.6_4-1.el7.x86_64.rpm MariaDB-shared-10.5.6_4-1.el7.x86_64.rpm MariaDB-backup-10.5.6_4-1.el7.x86_64.rpm
$ yum install galera-enterprise-4-26.4.5-1.el7.8.x86_64.rpm
$ yum install MariaDB-server-10.5.6_4-1.el7.x86_64.rpm
$ yum install MariaDB-columnstore-engine-10.5.6_4.5.4.1-1.el7.x86_64.rpm

$ pip install boto-2.49.0-py2.py3-none-any.whl
```



#### CMAPI Setup

MariaDB ColumnStore post-installation scripts fails if they find MariaDB Enterprise Server running on the system. Stop the Server and disable the service after installing the packages:

```bash
$ mkdir /opt/cmapi
$ chmod 755 /opt/cmapi
$ cp /tmp/mariadb-columnstore-cmapi-1.1.tar.gz /opt/cmapi
$ cd /opt/cmapi
$ tar -zxvf mariadb-columnstore-cmapi-1.1.tar.gz
$ ./service.sh install
```



## 4) Configuration

### MariaDB Server Configuration

MariaDB Enterprise Servers can be configured in the following ways:

+ System variables and options can be set in a configuration file (such as /etc/my.cnf). MariaDB Enterprise Server must be restarted to apply changes made to the configuration file.
+ System variables and options can be set on the command-line.
+ If a system variable supports dynamic changes, t hen it can be set on-the-fly using `SET` statement

For example, to configure MariaDB Enterprise Server via a configuration file:

```bash
$ vi /etc/my.cnf.d/columnstore.cnf
...
[mariadb]
server_id = 1
character_set_server = utf8
collation_server = utf8_general_ci
log_bin = /var/lib/mysql/mariadb-bin
log_bin_index = /var/lib/mysql/mariadb-bin.index
relay_log = /var/lib/mysql/mariadb-relay
relay_log_index = /var/lib/mysql/mariadb-relay.index
log_slave_updates = ON
gtid_strict_mode = ON
log_error = /var/lib/mysql/mariadb-error.log
columnstore_use_import_for_batchinsert=ALWAYS
...
```



### Storage Manager Configuration For S3

MariaDB ColumnStore can use any object store that is Amazon S3 API compatible. ColumnStore's `Storage Manager` uses a persistent disk cache for read/write operations so that it has minimal performance impact on ColumnStore. In some cases it will perform better than local disk operations.

Edit the `/etc/columnstore/storagemanager.cnf` file to configure Amazon S3 modify `[ObjectStorage]`, `[S3]` sections: 

```bash
[ObjectStorage]
service = S3

[S3]
region = <S3-Region>
bucket = <S3-Bucket-Name>
endpoint = <your_s3_endpoint>
aws_access_key_id = <AWS-Access-Key-ID>
aws_secret_access_key = <AWS-Secret-Access-Key>

[Cache]
cache_size = <your_local_cache_size>
path = <your_local_cache_path>
```

- The default local cache size is 2 GB.
- The default local cache path is `/var/lib/columnstore/storagemanager/cache`.



### DNS Settings

MariaDB ColumnStore's CMAPI requires all nodes to have host names that are resolvable on all other nodes. If your infrastructure does not configure DNS mentally, then you may need to configure static DNS entries in each server's `/etc/hosts` file.

```bash
10.0.0.1		cs1
10.0.0.11		cs2
10.0.0.111	cs3
```



### Cross Engine Joins

When a cross engine join is executed, the `ExeMgr` process connects to the server using the `root` user with no password by default. MariaDB Enterprise Server 10.5 will reject this login attempt by default. If you plan to use Cross Engine Joins, you need to configure ColumnStore to use a different user account and password.

```bash
$ sudo mcsSetConfig CrossEngineSupport Host 127.0.0.1
$ sudo mcsSetConfig CrossEngineSupport Port 3306
$ sudo mcsSetConfig CrossEngineSupport User <cross_engine_user>
$ sudo mcsSetConfig CrossEngineSupport Password <cross_engine_passwd>
```

The `cross_engine@127.0.0.1` user account needs to be created on the server after it has been started.



## 5) Post-Installation

### Server Start

```bash
$ sudo systemctl start mariadb
$ sudo systemctl enable mariadb

$ sudo systemctl start mariadb-columnstore
$ sudo systemctl enable mariadb-columnstore

$ sudo systemctl start mariadb-columnstore-cmapi
$ sudo systemctl enable mariadb-columnstore-cmapi
```



### Configure CMAPI API key

Create an API key for the cluster.

This API key should be stored securely and kept confidential, because it can be used to add nodes to the multi-node ColumnStore deployment.

For example, to create a random 256-bit API key using `openssl rand`:

```bash
$ openssl rand -hex 32
8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55
```

Set the API key for the cluster by using `curl` to send the `status` command to CMAPI Server.

The new API key needs to be provided as part of the `X-API-key` HTML header.

The output can be piped to `jq`, so that the JSON results are pretty-printed.

```bash
$ curl -k -s https://cs1:8640/cmapi/0.4.0/cluster/status --header 'Content-Type:application/json' --header 'x-api-key:8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55' | jq .
```



### Configure Cluster 

From CMAPI 1.1, you need to add all the nodes even the `PM1` node.

```bash
# cs1 Node Add
curl -k -s -X PUT https://10.0.0.86:8640/cmapi/0.4.0/cluster/add-node --header 'Content-Type:application/json' --header 'x-api-key:8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55' --data '{"timeout":120, "node": "cs1"}' | jq .
   
# cs2 Node Add
curl -k -s -X PUT https://10.0.0.86:8640/cmapi/0.4.0/cluster/add-node --header 'Content-Type:application/json' --header 'x-api-key:8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55' --data '{"timeout":120, "node": "cs2"}' | jq .

# cs3 Node Add
curl -k -s -X PUT https://10.0.0.86:8640/cmapi/0.4.0/cluster/add-node --header 'Content-Type:application/json' --header 'x-api-key:8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55' --data '{"timeout":120, "node": "cs3"}' | jq .
```



After the successful 3 node addition, you can have status like the follows:

```bash
curl -k -s https://10.0.0.86:8640/cmapi/0.4.0/cluster/status --header 'Content-Type:application/json' --header 'x-api-key:8520b9ff570421cc95077a50da660499a1238ffe88a488075efeb0f15b31fe55' | jq .
{
  "timestamp": "2020-10-27 07:17:43.902533",
  "cs1": {
    "timestamp": "2020-10-27 07:17:43.909381",
    "uptime": 79566,
    "dbrm_mode": "master",
    "cluster_mode": "readwrite",
    "dbroots": [
      "1"
    ],
    "module_id": 1,
    "services": [
      {
        "name": "workernode",
        "pid": 29821
      },
      {
        "name": "controllernode",
        "pid": 29879
      },
      {
        "name": "PrimProc",
        "pid": 29886
      },
      {
        "name": "ExeMgr",
        "pid": 29923
      },
      {
        "name": "WriteEngine",
        "pid": 29934
      },
      {
        "name": "DMLProc",
        "pid": 29982
      },
      {
        "name": "DDLProc",
        "pid": 29985
      }
    ]
  },
  "cs2": {
    "timestamp": "2020-10-27 07:17:43.947737",
    "uptime": 79561,
    "dbrm_mode": "slave",
    "cluster_mode": "readonly",
    "dbroots": [
      "2"
    ],
    "module_id": 2,
    "services": [
      {
        "name": "workernode",
        "pid": 25328
      },
      {
        "name": "PrimProc",
        "pid": 25334
      },
      {
        "name": "ExeMgr",
        "pid": 25375
      },
      {
        "name": "WriteEngine",
        "pid": 25387
      }
    ]
  },
  "cs3": {
    "timestamp": "2020-10-27 07:17:43.985964",
    "uptime": 505842,
    "dbrm_mode": "slave",
    "cluster_mode": "readonly",
    "dbroots": [
      "3"
    ],
    "module_id": 3,
    "services": [
      {
        "name": "workernode",
        "pid": 31327
      },
      {
        "name": "PrimProc",
        "pid": 31333
      },
      {
        "name": "ExeMgr",
        "pid": 31373
      },
      {
        "name": "WriteEngine",
        "pid": 31384
      }
    ]
  },
  "num_nodes": 3
}
```



### Creating System Table (Optional)

If you changed StorageManager Configuration (/etc/columnstore/storagemanager.cnf) after package installation, for example, `Service=local` to `Service=S3`, then you might need to create system catalog again.

```bash
# dbbuilder 7
```

If your system tables already exist, then you will see a message like the following:

```
Build system catalog was not completed. System catalog appears to exist.  It will remain intact for reuse.  The database is not recreated.
```



### Cross Engine Join User

```mariadb
CREATE USER 'cross_engine'@'127.0.0.1' IDENTIFIED BY "cross_engine_passwd";
GRANT SELECT ON *.* TO 'cross_engine'@'127.0.0.1';
```



### Replication User

```mariadb
CREATE USER 'repl'@'192.0.2.%' IDENTIFIED BY 'passwd';
GRANT REPLICATION SLAVE, BINLOG MONITOR ON *.* TO 'repl'@'192.0.2.%';
```



### Configure MariaDB Replication

Connect to the replica server.

```bash
$ sudo mariadb
```

Configure replication on all the replica servers.

```mariadb
CHANGE MASTER TO
	  master_host='10.0.0.86', 
    master_user='repl',
    master_password='Passwd';
    
CHANGE MASTER TO master_use_gtid=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G

SET GLOBAL read_only=ON;
```



## 6) Administration

### MariaDB Enterprise Server:

| **Operation**          | **Command**                      |
| ---------------------- | -------------------------------- |
| Start                  | `sudo systemctl start mariadb`   |
| Stop                   | `sudo systemctl stop mariadb`    |
| Restart                | `sudo systemctl restart mariadb` |
| Enable during startup  | `sudo systemctl enable mariadb`  |
| Disable during startup | `sudo systemctl disable mariadb` |
| Status                 | `sudo systemctl status mariadb`  |

### MariaDB ColumnStore:

| **Operation**          | **Command**                                  |
| ---------------------- | -------------------------------------------- |
| Start                  | `sudo systemctl start mariadb-columnstore`   |
| Stop                   | `sudo systemctl stop mariadb-columnstore`    |
| Restart                | `sudo systemctl restart mariadb-columnstore` |
| Enable during startup  | `sudo systemctl enable mariadb-columnstore`  |
| Disable during startup | `sudo systemctl disable mariadb-columnstore` |
| Status                 | `sudo systemctl status mariadb-columnstore`  |

### MariaDB ColumnStore CMAPI:

| **Operation**          | **Command**                                        |
| ---------------------- | -------------------------------------------------- |
| Start                  | `sudo systemctl start mariadb-columnstore-cmapi`   |
| Stop                   | `sudo systemctl stop mariadb-columnstore-cmapi`    |
| Restart                | `sudo systemctl restart mariadb-columnstore-cmapi` |
| Enable during startup  | `sudo systemctl enable mariadb-columnstore-cmapi`  |
| Disable during startup | `sudo systemctl disable mariadb-columnstore-cmapi` |
| Status                 | `sudo systemctl status mariadb-columnstore-cmapi`  |

### Examples using curl:

#### Get Status:

```bash
curl -k -s https://<pm1_ip>:8640/cmapi/0.4.0/cluster/status --header 'Content-Type:application/json' --header 'x-api-key:somekey123' | jq .
```

#### Start Cluster:

```bash
curl -k -s -X PUT https://<pm1_ip>:8640/cmapi/0.4.0/cluster/start --header 'Content-Type:application/json' --header 'x-api-key:somekey123' --data '{"timeout":20}' | jq .
```

#### Stop Cluster:

```bash
curl -k -s -X PUT https://<pm1_ip>:8640/cmapi/0.4.0/cluster/shutdown --header 'Content-Type:application/json' --header 'x-api-key:somekey123' --data '{"timeout":20}' | jq .
```

#### Add Node:

```bash
curl -k -s -X PUT https://<pm1_ip>:8640/cmapi/0.4.0/cluster/add-node --header 'Content-Type:application/json' --header 'x-api-key:somekey123' --data '{"timeout":20, "node": "<pm2_ip>"}' | jq .
```

#### Remove Node:

```bash
curl -k -s -X PUT https://<pm1_ip>:8640/cmapi/0.4.0/cluster/remove-node --header 'Content-Type:application/json' --header 'x-api-key:MyAPIKey123' --data '{"timeout":20, "node": "<pm2_ip>"}' | jq .
```

