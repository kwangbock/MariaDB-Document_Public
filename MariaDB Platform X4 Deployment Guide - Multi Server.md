# MariaDB Platform X4 Deployment Guide - Multi Server

MariaDB Platform X4 provides a single database stack that delivers cloud-native storage and smart transactions on a single platform.

Platform X4 incorporates MariaDB Enterprise Server 10.4, MariaDB MaxScale 2.4, and MariaDB ColumnStore 1.4.



**Table of Contents**

[TOC]

## 1) Assumption

##### +++ Architecture Image



+ 2 Node Deployment on RHEL7.7

+ non-root user guidance
+ Utilize **<u>AWS S3</u>** for ColumnStore Storage



## 2) Prerequisites / System Preparation

MariaDB Platform X4 utilizes MariaDB ColumnStore and its installation for multi-node deployments are configured from the initial Performance Module. For these post-installation scripts to connect to and configure the other Servers, systems hosting ColumnStore Instances require some additional configuration.



### Firewalls and SELinux

MariaDB ColumnStore requires port 3306, 8600 to 8650, 8700, and 8800. On some operating systems, this may require configuring the firewall and SELinux to permit TCP traffic over these ports.

The `postConfigure` script runs on the initial Performance Module and from this system connects over the network to all other Performance Modules to configure them to operate as back-end storage to MariaDB ColumnStore. To avoid confusion and potential problems, disable the firewall and put SELinux in permissive mode.

```bash
$ sudo getenforce
$ sestatus

$ sudo setenforce 0
$ sudo vi /etc/selinux/config
SELINUX=permissive|disabled

## Allow SSH daemon to run on port 2022
# sudo yum install policycoreutils-python

$ semanage port -l | grep ssh
$ semanage port -a -t ssh_port_t -p tcp 2022	# add port
$ semanage port -d -t ssh_port_t -p tcp 2022	# delete port
```



### SSH Configuration

The `postConfigure` script uses SSH to connect from the initial Performance Module (which ColumnStore identifies internally as pm1) to the other Servers in the deployment. For pm1 to connect to the other Servers, you need to generate an SSH key and share it with the other hosts in the deployments for password-less SSH. Otherwise all nodes must have the same password during installation.



#### How to setup SSH equivalence, password-less SSH

First, generate the SSH key:

```bash
$ sudo ssh-keygen
```

Then, copy the SSH key to each Server in your deployment:

``` bash
$ sudo ssh-copy-id -i ~/.ssh/id_rsa.pub 172.17.0.2
```

Once this is done, restart the SSH daemon on each Server:

```bash
$ sudo systemctl restart sshd
```

You can test the connection using the SSH client from the initial system to connect to the others:

```bash
$ ssh 172.17.0.2
```



#### SSH Daemon Configuration

```bash
$ sudo vi /etc/ssh/sshd_config
...
PasswordAuthentication yes
PermitRootLogin yes
AllowUsers root
AllowGroups root
DenyUsers
DenyGroups
...
```



### Storage Manager

MariaDB ColumnStore manages back-end storage using Performance Modules. When a MariaDB Enterprise Server goes down, the deployment loses access to the data stored in the local Performance Module.

Storage managers allow MariaDB ColumnStore to store data in network accessible file systems. When using a storage manager, the deployment can access a shared pool of Performance Modules, which remain accessible even if the Server goes down.

If you would like to use any sort of storage manager, you need to install and configure it prior to installing MariaDB ColumnStore.

MariaDB ColumnStore 1.4 supports three forms of external storage, configured by `postConfigure` script:

| **Storage** | **Description**                                              |
| ----------- | ------------------------------------------------------------ |
| External    | Mounted directory in the local file system, shared by all Servers. |
| GlusterFS   | Network attached storage file system.                        |
| Amazon S3   | Simple Storage Service from AWS, provides object storage to each Server for MariaDB ColumnStore. |



### Character Encoding

In multi-node modek Performance Modules are shared by all MariaDB Enterprise Servers in the deployment. Different systems and especially systems in different parts of the world or using different cloud platforms, may have different locale settings, which can lead to unexpected results when a Server writes data using one character encoding that is in turn read by a Server expecting a different character encoding.

To avoid conflicts, set the locale on all systems to the same value:

```bash 
$ sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```



### Dependent Packages

```bash
$ sudo yum -y install epel-release
$ sudo yum -y install wget

### If you are installing via YUM (Repository)
$ sudo yum -y install nmap jemalloc

### If you are installing via Package Tarball
$ sudo yum -y install boost expect perl perl-DBI openssl zlib file sudo libaio rsync snappy net-tools numactl-libs nmap jemalloc
```





## 3) MariaDB Platform X4 Installation

Available deployment methods are component-specific: (<u>**As of Mar 2020**</u>)

| **Deployment Method**                                        | **ES10.4** | **ColumnStore 1.4** | **MaxScale 2.4** |
| ------------------------------------------------------------ | ---------- | ------------------- | ---------------- |
| [Repository](https://mariadb.com/docs/deploy/installation/#install-repository) | Yes        | Yes                 | Yes              |
| [Repository Mirror](https://mariadb.com/docs/deploy/installation/#install-repomirror) | Yes        | Yes                 | Yes              |
| [Binary Tarball](https://mariadb.com/docs/deploy/installation/#install-bintar) | Yes        | **No**              | Yes              |
| [Package Tarball](https://mariadb.com/docs/deploy/installation/#install-packagetar) | Yes        | Yes                 | **No**           |
| [MariaDB SkySQL](https://mariadb.com/docs/deploy/installation/#install-skysql) | Yes        | Yes                 | Yes              |



### Install via YUM (Repository)

Retrieve your Customer Download Token at *https://customers.mariadb.com/downloads/token/* and substitute for `customer_download_token` in the following directions.

MariaDB ColumnStore 1.4 is available on MariaDB Enterprise 10.4.



To configure YUM package repositories:

```bash
$ sudo yum install wget
$ wget https://dlm.mariadb.com/enterprise-release-helpers/mariadb_es_repo_setup

$ echo "aed539ac1ad8db2b6ebb9791ae30a9ebfd9c00d91c322fc295ee56ae7fe414e8  mariadb_es_repo_setup" \
    | sha256sum -c -

$ chmod +x mariadb_es_repo_setup

$ sudo ./mariadb_es_repo_setup --token="customer_download_token" --apply \
   --mariadb-server-version="10.4"
```

To install MariaDB Platform X4 and package dependencies on RHEL 7 and CentOS 7:

```bash
$ sudo yum install MariaDB-server \
    MariaDB-columnstore-platform MariaDB-columnstore-engine
```



Installation Log:

```bash
[root@xdb1 ~]# yum install MariaDB-server MariaDB-columnstore-platform MariaDB-columnstore-engine
Loaded plugins: amazon-id, search-disabled-repos
mariadb-es-main                                                                                                                                | 2.9 kB  00:00:00
mariadb-maxscale                                                                                                                               | 2.4 kB  00:00:00
mariadb-tools                                                                                                                                  | 2.9 kB  00:00:00
(1/3): mariadb-maxscale/7Server/x86_64/primary_db                                                                                              | 6.9 kB  00:00:01
(2/3): mariadb-tools/7Server/x86_64/primary_db                                                                                                 |  13 kB  00:00:01
(3/3): mariadb-es-main/primary_db                                                                                                              |  28 kB  00:00:01
Resolving Dependencies
--> Running transaction check
---> Package MariaDB-columnstore-engine.x86_64 0:10.4.12_6-1.el7 will be installed
--> Processing Dependency: MariaDB-common for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_date_time-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libdmlpackage.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libquerytele.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libidbdatafile.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_thread-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_regex-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libcompress.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libconfigcpp.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libwindowfunction.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libmessageqcpp.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_system-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: liblibmysql_client.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libcacheutils.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libbrm.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_chrono-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_atomic-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libfuncexp.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libjoiner.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libcommon.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libdataconvert.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libwriteengineclient.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libexecplan.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libloggingcpp.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libddlpackageproc.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libthrift.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: librowgroup.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libalarmmanager.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libudfsdk.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: liboamcpp.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libthreadpool.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libregr.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libboost_filesystem-mt.so.1.53.0()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libwriteengine.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libddlpackage.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: librwlock.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libdmlpackageproc.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libquerystats.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libjoblist.so()(64bit) for package: MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64
---> Package MariaDB-columnstore-platform.x86_64 0:10.4.12_6-1.el7 will be installed
--> Processing Dependency: /usr/bin/expect for package: MariaDB-columnstore-platform-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libmariadb.so.3()(64bit) for package: MariaDB-columnstore-platform-10.4.12_6-1.el7.x86_64
---> Package MariaDB-server.x86_64 0:10.4.12_6-1.el7 will be installed
--> Processing Dependency: perl(Sys::Hostname) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(DBI) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: MariaDB-client for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(POSIX) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: galera-enterprise-4 for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libaio.so.1(LIBAIO_0.4)(64bit) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(Data::Dumper) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: /usr/bin/perl for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(File::Path) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(Getopt::Long) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(strict) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(File::Copy) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(File::Basename) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(vars) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: lsof for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libaio.so.1(LIBAIO_0.1)(64bit) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: perl(File::Temp) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Processing Dependency: libaio.so.1()(64bit) for package: MariaDB-server-10.4.12_6-1.el7.x86_64
--> Running transaction check
---> Package MariaDB-client.x86_64 0:10.4.12_6-1.el7 will be installed
--> Processing Dependency: perl(Exporter) for package: MariaDB-client-10.4.12_6-1.el7.x86_64
---> Package MariaDB-columnstore-libs.x86_64 0:10.4.12_6-1.el7 will be installed
---> Package MariaDB-common.x86_64 0:10.4.12_6-1.el7 will be installed
--> Processing Dependency: MariaDB-compat for package: MariaDB-common-10.4.12_6-1.el7.x86_64
---> Package MariaDB-shared.x86_64 0:10.4.12_6-1.el7 will be installed
---> Package boost-atomic.x86_64 0:1.53.0-27.el7 will be installed
---> Package boost-chrono.x86_64 0:1.53.0-27.el7 will be installed
---> Package boost-date-time.x86_64 0:1.53.0-27.el7 will be installed
---> Package boost-filesystem.x86_64 0:1.53.0-27.el7 will be installed
---> Package boost-regex.x86_64 0:1.53.0-27.el7 will be installed
--> Processing Dependency: libicuuc.so.50()(64bit) for package: boost-regex-1.53.0-27.el7.x86_64
--> Processing Dependency: libicui18n.so.50()(64bit) for package: boost-regex-1.53.0-27.el7.x86_64
--> Processing Dependency: libicudata.so.50()(64bit) for package: boost-regex-1.53.0-27.el7.x86_64
---> Package boost-system.x86_64 0:1.53.0-27.el7 will be installed
---> Package boost-thread.x86_64 0:1.53.0-27.el7 will be installed
---> Package expect.x86_64 0:5.45-14.el7_1 will be installed
--> Processing Dependency: libtcl8.5.so()(64bit) for package: expect-5.45-14.el7_1.x86_64
---> Package galera-enterprise-4.x86_64 0:26.4.4-1.rhel7.5.el7 will be installed
--> Processing Dependency: libboost_program_options.so.1.53.0()(64bit) for package: galera-enterprise-4-26.4.4-1.rhel7.5.el7.x86_64
---> Package libaio.x86_64 0:0.3.109-13.el7 will be installed
---> Package lsof.x86_64 0:4.87-6.el7 will be installed
---> Package perl.x86_64 4:5.16.3-294.el7_6 will be installed
--> Processing Dependency: perl-libs = 4:5.16.3-294.el7_6 for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Socket) >= 1.3 for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Scalar::Util) >= 1.10 for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl-macros for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl-libs for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(threads::shared) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(threads) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(constant) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Time::Local) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Time::HiRes) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Storable) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Socket) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Scalar::Util) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Pod::Simple::XHTML) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Pod::Simple::Search) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Filter::Util::Call) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(File::Spec::Unix) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(File::Spec::Functions) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(File::Spec) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Cwd) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: perl(Carp) for package: 4:perl-5.16.3-294.el7_6.x86_64
--> Processing Dependency: libperl.so()(64bit) for package: 4:perl-5.16.3-294.el7_6.x86_64
---> Package perl-DBI.x86_64 0:1.627-4.el7 will be installed
--> Processing Dependency: perl(RPC::PlServer) >= 0.2001 for package: perl-DBI-1.627-4.el7.x86_64
--> Processing Dependency: perl(RPC::PlClient) >= 0.2000 for package: perl-DBI-1.627-4.el7.x86_64
---> Package perl-Data-Dumper.x86_64 0:2.145-3.el7 will be installed
---> Package perl-File-Path.noarch 0:2.09-2.el7 will be installed
---> Package perl-File-Temp.noarch 0:0.23.01-3.el7 will be installed
---> Package perl-Getopt-Long.noarch 0:2.40-3.el7 will be installed
--> Processing Dependency: perl(Pod::Usage) >= 1.14 for package: perl-Getopt-Long-2.40-3.el7.noarch
--> Processing Dependency: perl(Text::ParseWords) for package: perl-Getopt-Long-2.40-3.el7.noarch
--> Running transaction check
---> Package MariaDB-compat.x86_64 0:10.4.12_6-1.el7 will be obsoleting
---> Package boost-program-options.x86_64 0:1.53.0-27.el7 will be installed
---> Package libicu.x86_64 0:50.2-4.el7_7 will be installed
---> Package mariadb-libs.x86_64 1:5.5.64-1.el7 will be obsoleted
---> Package perl-Carp.noarch 0:1.26-244.el7 will be installed
---> Package perl-Exporter.noarch 0:5.68-3.el7 will be installed
---> Package perl-Filter.x86_64 0:1.49-3.el7 will be installed
---> Package perl-PathTools.x86_64 0:3.40-5.el7 will be installed
---> Package perl-PlRPC.noarch 0:0.2020-14.el7 will be installed
--> Processing Dependency: perl(Net::Daemon) >= 0.13 for package: perl-PlRPC-0.2020-14.el7.noarch
--> Processing Dependency: perl(Net::Daemon::Test) for package: perl-PlRPC-0.2020-14.el7.noarch
--> Processing Dependency: perl(Net::Daemon::Log) for package: perl-PlRPC-0.2020-14.el7.noarch
--> Processing Dependency: perl(Compress::Zlib) for package: perl-PlRPC-0.2020-14.el7.noarch
---> Package perl-Pod-Simple.noarch 1:3.28-4.el7 will be installed
--> Processing Dependency: perl(Pod::Escapes) >= 1.04 for package: 1:perl-Pod-Simple-3.28-4.el7.noarch
--> Processing Dependency: perl(Encode) for package: 1:perl-Pod-Simple-3.28-4.el7.noarch
---> Package perl-Pod-Usage.noarch 0:1.63-3.el7 will be installed
--> Processing Dependency: perl(Pod::Text) >= 3.15 for package: perl-Pod-Usage-1.63-3.el7.noarch
--> Processing Dependency: perl-Pod-Perldoc for package: perl-Pod-Usage-1.63-3.el7.noarch
---> Package perl-Scalar-List-Utils.x86_64 0:1.27-248.el7 will be installed
---> Package perl-Socket.x86_64 0:2.010-4.el7 will be installed
---> Package perl-Storable.x86_64 0:2.45-3.el7 will be installed
---> Package perl-Text-ParseWords.noarch 0:3.29-4.el7 will be installed
---> Package perl-Time-HiRes.x86_64 4:1.9725-3.el7 will be installed
---> Package perl-Time-Local.noarch 0:1.2300-2.el7 will be installed
---> Package perl-constant.noarch 0:1.27-2.el7 will be installed
---> Package perl-libs.x86_64 4:5.16.3-294.el7_6 will be installed
---> Package perl-macros.x86_64 4:5.16.3-294.el7_6 will be installed
---> Package perl-threads.x86_64 0:1.87-4.el7 will be installed
---> Package perl-threads-shared.x86_64 0:1.43-6.el7 will be installed
---> Package tcl.x86_64 1:8.5.13-8.el7 will be installed
--> Running transaction check
---> Package perl-Encode.x86_64 0:2.51-7.el7 will be installed
---> Package perl-IO-Compress.noarch 0:2.061-2.el7 will be installed
--> Processing Dependency: perl(Compress::Raw::Zlib) >= 2.061 for package: perl-IO-Compress-2.061-2.el7.noarch
--> Processing Dependency: perl(Compress::Raw::Bzip2) >= 2.061 for package: perl-IO-Compress-2.061-2.el7.noarch
---> Package perl-Net-Daemon.noarch 0:0.48-5.el7 will be installed
---> Package perl-Pod-Escapes.noarch 1:1.04-294.el7_6 will be installed
---> Package perl-Pod-Perldoc.noarch 0:3.20-4.el7 will be installed
--> Processing Dependency: perl(parent) for package: perl-Pod-Perldoc-3.20-4.el7.noarch
--> Processing Dependency: perl(HTTP::Tiny) for package: perl-Pod-Perldoc-3.20-4.el7.noarch
---> Package perl-podlators.noarch 0:2.5.1-3.el7 will be installed
--> Running transaction check
---> Package perl-Compress-Raw-Bzip2.x86_64 0:2.061-3.el7 will be installed
---> Package perl-Compress-Raw-Zlib.x86_64 1:2.061-4.el7 will be installed
---> Package perl-HTTP-Tiny.noarch 0:0.033-3.el7 will be installed
---> Package perl-parent.noarch 1:0.225-244.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================================================================
 Package                                         Arch                      Version                                   Repository                                  Size
======================================================================================================================================================================
Installing:
 MariaDB-columnstore-engine                      x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            453 k
 MariaDB-columnstore-platform                    x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            2.6 M
 MariaDB-compat                                  x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            2.2 M
     replacing  mariadb-libs.x86_64 1:5.5.64-1.el7
 MariaDB-server                                  x86_64                    10.4.12_6-1.el7                           mariadb-es-main                             21 M
Installing for dependencies:
 MariaDB-client                                  x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            6.4 M
 MariaDB-columnstore-libs                        x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            3.4 M
 MariaDB-common                                  x86_64                    10.4.12_6-1.el7                           mariadb-es-main                             82 k
 MariaDB-shared                                  x86_64                    10.4.12_6-1.el7                           mariadb-es-main                            113 k
 boost-atomic                                    x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     35 k
 boost-chrono                                    x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     44 k
 boost-date-time                                 x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     52 k
 boost-filesystem                                x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     68 k
 boost-program-options                           x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                    156 k
 boost-regex                                     x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                    300 k
 boost-system                                    x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     40 k
 boost-thread                                    x86_64                    1.53.0-27.el7                             rhel-7-server-rhui-rpms                     57 k
 expect                                          x86_64                    5.45-14.el7_1                             rhel-7-server-rhui-rpms                    262 k
 galera-enterprise-4                             x86_64                    26.4.4-1.rhel7.5.el7                      mariadb-es-main                            9.9 M
 libaio                                          x86_64                    0.3.109-13.el7                            rhel-7-server-rhui-rpms                     24 k
 libicu                                          x86_64                    50.2-4.el7_7                              rhel-7-server-rhui-rpms                    6.9 M
 lsof                                            x86_64                    4.87-6.el7                                rhel-7-server-rhui-rpms                    331 k
 perl                                            x86_64                    4:5.16.3-294.el7_6                        rhel-7-server-rhui-rpms                    8.0 M
 perl-Carp                                       noarch                    1.26-244.el7                              rhel-7-server-rhui-rpms                     19 k
 perl-Compress-Raw-Bzip2                         x86_64                    2.061-3.el7                               rhel-7-server-rhui-rpms                     32 k
 perl-Compress-Raw-Zlib                          x86_64                    1:2.061-4.el7                             rhel-7-server-rhui-rpms                     57 k
 perl-DBI                                        x86_64                    1.627-4.el7                               rhel-7-server-rhui-rpms                    802 k
 perl-Data-Dumper                                x86_64                    2.145-3.el7                               rhel-7-server-rhui-rpms                     47 k
 perl-Encode                                     x86_64                    2.51-7.el7                                rhel-7-server-rhui-rpms                    1.5 M
 perl-Exporter                                   noarch                    5.68-3.el7                                rhel-7-server-rhui-rpms                     28 k
 perl-File-Path                                  noarch                    2.09-2.el7                                rhel-7-server-rhui-rpms                     27 k
 perl-File-Temp                                  noarch                    0.23.01-3.el7                             rhel-7-server-rhui-rpms                     56 k
 perl-Filter                                     x86_64                    1.49-3.el7                                rhel-7-server-rhui-rpms                     76 k
 perl-Getopt-Long                                noarch                    2.40-3.el7                                rhel-7-server-rhui-rpms                     56 k
 perl-HTTP-Tiny                                  noarch                    0.033-3.el7                               rhel-7-server-rhui-rpms                     38 k
 perl-IO-Compress                                noarch                    2.061-2.el7                               rhel-7-server-rhui-rpms                    260 k
 perl-Net-Daemon                                 noarch                    0.48-5.el7                                rhel-7-server-rhui-rpms                     51 k
 perl-PathTools                                  x86_64                    3.40-5.el7                                rhel-7-server-rhui-rpms                     83 k
 perl-PlRPC                                      noarch                    0.2020-14.el7                             rhel-7-server-rhui-rpms                     36 k
 perl-Pod-Escapes                                noarch                    1:1.04-294.el7_6                          rhel-7-server-rhui-rpms                     51 k
 perl-Pod-Perldoc                                noarch                    3.20-4.el7                                rhel-7-server-rhui-rpms                     87 k
 perl-Pod-Simple                                 noarch                    1:3.28-4.el7                              rhel-7-server-rhui-rpms                    216 k
 perl-Pod-Usage                                  noarch                    1.63-3.el7                                rhel-7-server-rhui-rpms                     27 k
 perl-Scalar-List-Utils                          x86_64                    1.27-248.el7                              rhel-7-server-rhui-rpms                     36 k
 perl-Socket                                     x86_64                    2.010-4.el7                               rhel-7-server-rhui-rpms                     49 k
 perl-Storable                                   x86_64                    2.45-3.el7                                rhel-7-server-rhui-rpms                     77 k
 perl-Text-ParseWords                            noarch                    3.29-4.el7                                rhel-7-server-rhui-rpms                     14 k
 perl-Time-HiRes                                 x86_64                    4:1.9725-3.el7                            rhel-7-server-rhui-rpms                     45 k
 perl-Time-Local                                 noarch                    1.2300-2.el7                              rhel-7-server-rhui-rpms                     24 k
 perl-constant                                   noarch                    1.27-2.el7                                rhel-7-server-rhui-rpms                     19 k
 perl-libs                                       x86_64                    4:5.16.3-294.el7_6                        rhel-7-server-rhui-rpms                    688 k
 perl-macros                                     x86_64                    4:5.16.3-294.el7_6                        rhel-7-server-rhui-rpms                     44 k
 perl-parent                                     noarch                    1:0.225-244.el7                           rhel-7-server-rhui-rpms                     12 k
 perl-podlators                                  noarch                    2.5.1-3.el7                               rhel-7-server-rhui-rpms                    112 k
 perl-threads                                    x86_64                    1.87-4.el7                                rhel-7-server-rhui-rpms                     49 k
 perl-threads-shared                             x86_64                    1.43-6.el7                                rhel-7-server-rhui-rpms                     39 k
 tcl                                             x86_64                    1:8.5.13-8.el7                            rhel-7-server-rhui-rpms                    1.9 M

Transaction Summary
======================================================================================================================================================================
Install  4 Packages (+52 Dependent packages)

Total download size: 69 M
Is this ok [y/d/N]: y
Downloading packages:
(1/56): MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64.rpm                                                                                  | 453 kB  00:00:02
(2/56): MariaDB-client-10.4.12_6-1.el7.x86_64.rpm                                                                                              | 6.4 MB  00:00:03
(3/56): MariaDB-columnstore-libs-10.4.12_6-1.el7.x86_64.rpm                                                                                    | 3.4 MB  00:00:01
(4/56): MariaDB-columnstore-platform-10.4.12_6-1.el7.x86_64.rpm                                                                                | 2.6 MB  00:00:01
(5/56): MariaDB-common-10.4.12_6-1.el7.x86_64.rpm                                                                                              |  82 kB  00:00:01
(6/56): MariaDB-compat-10.4.12_6-1.el7.x86_64.rpm                                                                                              | 2.2 MB  00:00:01
(7/56): MariaDB-server-10.4.12_6-1.el7.x86_64.rpm                                                                                              |  21 MB  00:00:02
(8/56): boost-chrono-1.53.0-27.el7.x86_64.rpm                                                                                                  |  44 kB  00:00:00
(9/56): boost-atomic-1.53.0-27.el7.x86_64.rpm                                                                                                  |  35 kB  00:00:00
(10/56): boost-date-time-1.53.0-27.el7.x86_64.rpm                                                                                              |  52 kB  00:00:00
(11/56): boost-filesystem-1.53.0-27.el7.x86_64.rpm                                                                                             |  68 kB  00:00:00
(12/56): boost-program-options-1.53.0-27.el7.x86_64.rpm                                                                                        | 156 kB  00:00:00
(13/56): boost-regex-1.53.0-27.el7.x86_64.rpm                                                                                                  | 300 kB  00:00:00
(14/56): boost-system-1.53.0-27.el7.x86_64.rpm                                                                                                 |  40 kB  00:00:00
(15/56): MariaDB-shared-10.4.12_6-1.el7.x86_64.rpm                                                                                             | 113 kB  00:00:00
(16/56): boost-thread-1.53.0-27.el7.x86_64.rpm                                                                                                 |  57 kB  00:00:00
(17/56): expect-5.45-14.el7_1.x86_64.rpm                                                                                                       | 262 kB  00:00:00
(18/56): libaio-0.3.109-13.el7.x86_64.rpm                                                                                                      |  24 kB  00:00:00
(19/56): lsof-4.87-6.el7.x86_64.rpm                                                                                                            | 331 kB  00:00:00
(20/56): libicu-50.2-4.el7_7.x86_64.rpm                                                                                                        | 6.9 MB  00:00:00
(21/56): perl-Carp-1.26-244.el7.noarch.rpm                                                                                                     |  19 kB  00:00:00
(22/56): perl-5.16.3-294.el7_6.x86_64.rpm                                                                                                      | 8.0 MB  00:00:00
(23/56): perl-Compress-Raw-Bzip2-2.061-3.el7.x86_64.rpm                                                                                        |  32 kB  00:00:00
(24/56): perl-Compress-Raw-Zlib-2.061-4.el7.x86_64.rpm                                                                                         |  57 kB  00:00:00
(25/56): perl-DBI-1.627-4.el7.x86_64.rpm                                                                                                       | 802 kB  00:00:00
(26/56): perl-Data-Dumper-2.145-3.el7.x86_64.rpm                                                                                               |  47 kB  00:00:00
(27/56): perl-Encode-2.51-7.el7.x86_64.rpm                                                                                                     | 1.5 MB  00:00:00
(28/56): perl-Exporter-5.68-3.el7.noarch.rpm                                                                                                   |  28 kB  00:00:00
(29/56): perl-File-Path-2.09-2.el7.noarch.rpm                                                                                                  |  27 kB  00:00:00
(30/56): perl-File-Temp-0.23.01-3.el7.noarch.rpm                                                                                               |  56 kB  00:00:00
(31/56): perl-Filter-1.49-3.el7.x86_64.rpm                                                                                                     |  76 kB  00:00:00
(32/56): perl-Getopt-Long-2.40-3.el7.noarch.rpm                                                                                                |  56 kB  00:00:00
(33/56): perl-HTTP-Tiny-0.033-3.el7.noarch.rpm                                                                                                 |  38 kB  00:00:00
(34/56): perl-IO-Compress-2.061-2.el7.noarch.rpm                                                                                               | 260 kB  00:00:00
(35/56): perl-Net-Daemon-0.48-5.el7.noarch.rpm                                                                                                 |  51 kB  00:00:00
(36/56): perl-PathTools-3.40-5.el7.x86_64.rpm                                                                                                  |  83 kB  00:00:00
(37/56): perl-PlRPC-0.2020-14.el7.noarch.rpm                                                                                                   |  36 kB  00:00:00
(38/56): perl-Pod-Escapes-1.04-294.el7_6.noarch.rpm                                                                                            |  51 kB  00:00:00
(39/56): perl-Pod-Perldoc-3.20-4.el7.noarch.rpm                                                                                                |  87 kB  00:00:00
(40/56): perl-Pod-Simple-3.28-4.el7.noarch.rpm                                                                                                 | 216 kB  00:00:00
(41/56): perl-Pod-Usage-1.63-3.el7.noarch.rpm                                                                                                  |  27 kB  00:00:00
(42/56): perl-Scalar-List-Utils-1.27-248.el7.x86_64.rpm                                                                                        |  36 kB  00:00:00
(43/56): perl-Socket-2.010-4.el7.x86_64.rpm                                                                                                    |  49 kB  00:00:00
(44/56): perl-Storable-2.45-3.el7.x86_64.rpm                                                                                                   |  77 kB  00:00:00
(45/56): perl-Text-ParseWords-3.29-4.el7.noarch.rpm                                                                                            |  14 kB  00:00:00
(46/56): perl-Time-HiRes-1.9725-3.el7.x86_64.rpm                                                                                               |  45 kB  00:00:00
(47/56): perl-Time-Local-1.2300-2.el7.noarch.rpm                                                                                               |  24 kB  00:00:00
(48/56): perl-constant-1.27-2.el7.noarch.rpm                                                                                                   |  19 kB  00:00:00
(49/56): perl-libs-5.16.3-294.el7_6.x86_64.rpm                                                                                                 | 688 kB  00:00:00
(50/56): perl-macros-5.16.3-294.el7_6.x86_64.rpm                                                                                               |  44 kB  00:00:00
(51/56): perl-parent-0.225-244.el7.noarch.rpm                                                                                                  |  12 kB  00:00:00
(52/56): perl-podlators-2.5.1-3.el7.noarch.rpm                                                                                                 | 112 kB  00:00:00
(53/56): perl-threads-1.87-4.el7.x86_64.rpm                                                                                                    |  49 kB  00:00:00
(54/56): perl-threads-shared-1.43-6.el7.x86_64.rpm                                                                                             |  39 kB  00:00:00
(55/56): tcl-8.5.13-8.el7.x86_64.rpm                                                                                                           | 1.9 MB  00:00:00
(56/56): galera-enterprise-4-26.4.4-1.rhel7.5.el7.x86_64.rpm                                                                                   | 9.9 MB  00:00:02
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                 7.0 MB/s |  69 MB  00:00:09
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : MariaDB-compat-10.4.12_6-1.el7.x86_64                                                                                                             1/57
  Installing : MariaDB-common-10.4.12_6-1.el7.x86_64                                                                                                             2/57
  Installing : boost-system-1.53.0-27.el7.x86_64                                                                                                                 3/57
  Installing : boost-filesystem-1.53.0-27.el7.x86_64                                                                                                             4/57
  Installing : boost-thread-1.53.0-27.el7.x86_64                                                                                                                 5/57
  Installing : boost-chrono-1.53.0-27.el7.x86_64                                                                                                                 6/57
  Installing : MariaDB-columnstore-libs-10.4.12_6-1.el7.x86_64                                                                                                   7/57
  Installing : boost-atomic-1.53.0-27.el7.x86_64                                                                                                                 8/57
  Installing : boost-date-time-1.53.0-27.el7.x86_64                                                                                                              9/57
  Installing : MariaDB-shared-10.4.12_6-1.el7.x86_64                                                                                                            10/57
  Installing : 1:perl-parent-0.225-244.el7.noarch                                                                                                               11/57
  Installing : perl-HTTP-Tiny-0.033-3.el7.noarch                                                                                                                12/57
  Installing : perl-podlators-2.5.1-3.el7.noarch                                                                                                                13/57
  Installing : perl-Pod-Perldoc-3.20-4.el7.noarch                                                                                                               14/57
  Installing : 1:perl-Pod-Escapes-1.04-294.el7_6.noarch                                                                                                         15/57
  Installing : perl-Encode-2.51-7.el7.x86_64                                                                                                                    16/57
  Installing : perl-Text-ParseWords-3.29-4.el7.noarch                                                                                                           17/57
  Installing : perl-Pod-Usage-1.63-3.el7.noarch                                                                                                                 18/57
  Installing : 4:perl-libs-5.16.3-294.el7_6.x86_64                                                                                                              19/57
  Installing : 4:perl-macros-5.16.3-294.el7_6.x86_64                                                                                                            20/57
  Installing : perl-Exporter-5.68-3.el7.noarch                                                                                                                  21/57
  Installing : perl-constant-1.27-2.el7.noarch                                                                                                                  22/57
  Installing : perl-Storable-2.45-3.el7.x86_64                                                                                                                  23/57
  Installing : perl-Time-Local-1.2300-2.el7.noarch                                                                                                              24/57
  Installing : 4:perl-Time-HiRes-1.9725-3.el7.x86_64                                                                                                            25/57
  Installing : perl-threads-1.87-4.el7.x86_64                                                                                                                   26/57
  Installing : perl-threads-shared-1.43-6.el7.x86_64                                                                                                            27/57
  Installing : perl-File-Path-2.09-2.el7.noarch                                                                                                                 28/57
  Installing : perl-PathTools-3.40-5.el7.x86_64                                                                                                                 29/57
  Installing : perl-File-Temp-0.23.01-3.el7.noarch                                                                                                              30/57
  Installing : perl-Scalar-List-Utils-1.27-248.el7.x86_64                                                                                                       31/57
  Installing : perl-Filter-1.49-3.el7.x86_64                                                                                                                    32/57
  Installing : perl-Socket-2.010-4.el7.x86_64                                                                                                                   33/57
  Installing : perl-Carp-1.26-244.el7.noarch                                                                                                                    34/57
  Installing : 1:perl-Pod-Simple-3.28-4.el7.noarch                                                                                                              35/57
  Installing : perl-Getopt-Long-2.40-3.el7.noarch                                                                                                               36/57
  Installing : 4:perl-5.16.3-294.el7_6.x86_64                                                                                                                   37/57
  Installing : perl-Data-Dumper-2.145-3.el7.x86_64                                                                                                              38/57
  Installing : MariaDB-client-10.4.12_6-1.el7.x86_64                                                                                                            39/57
  Installing : perl-Compress-Raw-Bzip2-2.061-3.el7.x86_64                                                                                                       40/57
  Installing : perl-Net-Daemon-0.48-5.el7.noarch                                                                                                                41/57
  Installing : 1:perl-Compress-Raw-Zlib-2.061-4.el7.x86_64                                                                                                      42/57
  Installing : perl-IO-Compress-2.061-2.el7.noarch                                                                                                              43/57
  Installing : perl-PlRPC-0.2020-14.el7.noarch                                                                                                                  44/57
  Installing : perl-DBI-1.627-4.el7.x86_64                                                                                                                      45/57
  Installing : libaio-0.3.109-13.el7.x86_64                                                                                                                     46/57
  Installing : boost-program-options-1.53.0-27.el7.x86_64                                                                                                       47/57
  Installing : galera-enterprise-4-26.4.4-1.rhel7.5.el7.x86_64                                                                                                  48/57
  Installing : libicu-50.2-4.el7_7.x86_64                                                                                                                       49/57
  Installing : boost-regex-1.53.0-27.el7.x86_64                                                                                                                 50/57
  Installing : lsof-4.87-6.el7.x86_64                                                                                                                           51/57
  Installing : 1:tcl-8.5.13-8.el7.x86_64                                                                                                                        52/57
  Installing : expect-5.45-14.el7_1.x86_64                                                                                                                      53/57
  Installing : MariaDB-columnstore-platform-10.4.12_6-1.el7.x86_64                                                                                              54/57
The next step on the node that will become PM1:

postConfigure


  Installing : MariaDB-server-10.4.12_6-1.el7.x86_64                                                                                                            55/57
2020-03-25  5:20:16 server_audit: MariaDB Audit Plugin version 2.0.2 STARTED.
2020-03-25  5:20:16 server_audit: Query cache is enabled with the TABLE events. Some table reads can be veiled.
2020-03-25  5:20:16 server_audit: STOPPED


Two all-privilege accounts were created.
One is root@localhost, it has no password, but you need to
be system 'root' user to connect. Use, for example, sudo mysql
The second is mysql@localhost, it has no password either, but
you need to be the system 'mysql' user to connect.
After connecting you can set the password, if you would need to be
able to connect as any of these users with a password and without sudo

See the MariaDB Knowledgebase at http://mariadb.com/kb or the
MySQL manual for more instructions.

As a MariaDB Corporation subscription customer please contact us
via https://support.mariadb.com/ to report problems.
You also can get consultative guidance on questions specific to your deployment,
such as how to tune for performance, high availability, security audits, and code review.

You also find detailed documentation about how to use MariaDB Enterprise Server at https://mariadb.com/docs/.
The latest information about MariaDB Server is available at https://mariadb.com/kb/en/library/release-notes/.

  Installing : MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64                                                                                                56/57
  Erasing    : 1:mariadb-libs-5.5.64-1.el7.x86_64                                                                                                               57/57
  Verifying  : galera-enterprise-4-26.4.4-1.rhel7.5.el7.x86_64                                                                                                   1/57
  Verifying  : boost-system-1.53.0-27.el7.x86_64                                                                                                                 2/57
  Verifying  : perl-HTTP-Tiny-0.033-3.el7.noarch                                                                                                                 3/57
  Verifying  : MariaDB-client-10.4.12_6-1.el7.x86_64                                                                                                             4/57
  Verifying  : boost-regex-1.53.0-27.el7.x86_64                                                                                                                  5/57
  Verifying  : perl-threads-shared-1.43-6.el7.x86_64                                                                                                             6/57
  Verifying  : 1:tcl-8.5.13-8.el7.x86_64                                                                                                                         7/57
  Verifying  : 1:perl-Pod-Escapes-1.04-294.el7_6.noarch                                                                                                          8/57
  Verifying  : lsof-4.87-6.el7.x86_64                                                                                                                            9/57
  Verifying  : perl-Exporter-5.68-3.el7.noarch                                                                                                                  10/57
  Verifying  : perl-constant-1.27-2.el7.noarch                                                                                                                  11/57
  Verifying  : perl-PathTools-3.40-5.el7.x86_64                                                                                                                 12/57
  Verifying  : perl-Storable-2.45-3.el7.x86_64                                                                                                                  13/57
  Verifying  : MariaDB-columnstore-platform-10.4.12_6-1.el7.x86_64                                                                                              14/57
  Verifying  : MariaDB-shared-10.4.12_6-1.el7.x86_64                                                                                                            15/57
  Verifying  : 1:perl-parent-0.225-244.el7.noarch                                                                                                               16/57
  Verifying  : perl-Compress-Raw-Bzip2-2.061-3.el7.x86_64                                                                                                       17/57
  Verifying  : perl-Net-Daemon-0.48-5.el7.noarch                                                                                                                18/57
  Verifying  : MariaDB-common-10.4.12_6-1.el7.x86_64                                                                                                            19/57
  Verifying  : 4:perl-libs-5.16.3-294.el7_6.x86_64                                                                                                              20/57
  Verifying  : perl-File-Temp-0.23.01-3.el7.noarch                                                                                                              21/57
  Verifying  : perl-Data-Dumper-2.145-3.el7.x86_64                                                                                                              22/57
  Verifying  : perl-Time-Local-1.2300-2.el7.noarch                                                                                                              23/57
  Verifying  : MariaDB-server-10.4.12_6-1.el7.x86_64                                                                                                            24/57
  Verifying  : libicu-50.2-4.el7_7.x86_64                                                                                                                       25/57
  Verifying  : boost-program-options-1.53.0-27.el7.x86_64                                                                                                       26/57
  Verifying  : perl-DBI-1.627-4.el7.x86_64                                                                                                                      27/57
  Verifying  : libaio-0.3.109-13.el7.x86_64                                                                                                                     28/57
  Verifying  : boost-filesystem-1.53.0-27.el7.x86_64                                                                                                            29/57
  Verifying  : 4:perl-macros-5.16.3-294.el7_6.x86_64                                                                                                            30/57
  Verifying  : 4:perl-5.16.3-294.el7_6.x86_64                                                                                                                   31/57
  Verifying  : boost-thread-1.53.0-27.el7.x86_64                                                                                                                32/57
  Verifying  : boost-date-time-1.53.0-27.el7.x86_64                                                                                                             33/57
  Verifying  : MariaDB-columnstore-engine-10.4.12_6-1.el7.x86_64                                                                                                34/57
  Verifying  : MariaDB-compat-10.4.12_6-1.el7.x86_64                                                                                                            35/57
  Verifying  : 4:perl-Time-HiRes-1.9725-3.el7.x86_64                                                                                                            36/57
  Verifying  : MariaDB-columnstore-libs-10.4.12_6-1.el7.x86_64                                                                                                  37/57
  Verifying  : 1:perl-Compress-Raw-Zlib-2.061-4.el7.x86_64                                                                                                      38/57
  Verifying  : perl-IO-Compress-2.061-2.el7.noarch                                                                                                              39/57
  Verifying  : perl-Pod-Usage-1.63-3.el7.noarch                                                                                                                 40/57
  Verifying  : perl-PlRPC-0.2020-14.el7.noarch                                                                                                                  41/57
  Verifying  : perl-Encode-2.51-7.el7.x86_64                                                                                                                    42/57
  Verifying  : boost-chrono-1.53.0-27.el7.x86_64                                                                                                                43/57
  Verifying  : perl-Pod-Perldoc-3.20-4.el7.noarch                                                                                                               44/57
  Verifying  : perl-podlators-2.5.1-3.el7.noarch                                                                                                                45/57
  Verifying  : perl-File-Path-2.09-2.el7.noarch                                                                                                                 46/57
  Verifying  : perl-threads-1.87-4.el7.x86_64                                                                                                                   47/57
  Verifying  : perl-Scalar-List-Utils-1.27-248.el7.x86_64                                                                                                       48/57
  Verifying  : expect-5.45-14.el7_1.x86_64                                                                                                                      49/57
  Verifying  : 1:perl-Pod-Simple-3.28-4.el7.noarch                                                                                                              50/57
  Verifying  : perl-Filter-1.49-3.el7.x86_64                                                                                                                    51/57
  Verifying  : perl-Getopt-Long-2.40-3.el7.noarch                                                                                                               52/57
  Verifying  : perl-Text-ParseWords-3.29-4.el7.noarch                                                                                                           53/57
  Verifying  : perl-Socket-2.010-4.el7.x86_64                                                                                                                   54/57
  Verifying  : perl-Carp-1.26-244.el7.noarch                                                                                                                    55/57
  Verifying  : boost-atomic-1.53.0-27.el7.x86_64                                                                                                                56/57
  Verifying  : 1:mariadb-libs-5.5.64-1.el7.x86_64                                                                                                               57/57

Installed:
  MariaDB-columnstore-engine.x86_64 0:10.4.12_6-1.el7       MariaDB-columnstore-platform.x86_64 0:10.4.12_6-1.el7       MariaDB-compat.x86_64 0:10.4.12_6-1.el7
  MariaDB-server.x86_64 0:10.4.12_6-1.el7

Dependency Installed:
  MariaDB-client.x86_64 0:10.4.12_6-1.el7           MariaDB-columnstore-libs.x86_64 0:10.4.12_6-1.el7           MariaDB-common.x86_64 0:10.4.12_6-1.el7
  MariaDB-shared.x86_64 0:10.4.12_6-1.el7           boost-atomic.x86_64 0:1.53.0-27.el7                         boost-chrono.x86_64 0:1.53.0-27.el7
  boost-date-time.x86_64 0:1.53.0-27.el7            boost-filesystem.x86_64 0:1.53.0-27.el7                     boost-program-options.x86_64 0:1.53.0-27.el7
  boost-regex.x86_64 0:1.53.0-27.el7                boost-system.x86_64 0:1.53.0-27.el7                         boost-thread.x86_64 0:1.53.0-27.el7
  expect.x86_64 0:5.45-14.el7_1                     galera-enterprise-4.x86_64 0:26.4.4-1.rhel7.5.el7           libaio.x86_64 0:0.3.109-13.el7
  libicu.x86_64 0:50.2-4.el7_7                      lsof.x86_64 0:4.87-6.el7                                    perl.x86_64 4:5.16.3-294.el7_6
  perl-Carp.noarch 0:1.26-244.el7                   perl-Compress-Raw-Bzip2.x86_64 0:2.061-3.el7                perl-Compress-Raw-Zlib.x86_64 1:2.061-4.el7
  perl-DBI.x86_64 0:1.627-4.el7                     perl-Data-Dumper.x86_64 0:2.145-3.el7                       perl-Encode.x86_64 0:2.51-7.el7
  perl-Exporter.noarch 0:5.68-3.el7                 perl-File-Path.noarch 0:2.09-2.el7                          perl-File-Temp.noarch 0:0.23.01-3.el7
  perl-Filter.x86_64 0:1.49-3.el7                   perl-Getopt-Long.noarch 0:2.40-3.el7                        perl-HTTP-Tiny.noarch 0:0.033-3.el7
  perl-IO-Compress.noarch 0:2.061-2.el7             perl-Net-Daemon.noarch 0:0.48-5.el7                         perl-PathTools.x86_64 0:3.40-5.el7
  perl-PlRPC.noarch 0:0.2020-14.el7                 perl-Pod-Escapes.noarch 1:1.04-294.el7_6                    perl-Pod-Perldoc.noarch 0:3.20-4.el7
  perl-Pod-Simple.noarch 1:3.28-4.el7               perl-Pod-Usage.noarch 0:1.63-3.el7                          perl-Scalar-List-Utils.x86_64 0:1.27-248.el7
  perl-Socket.x86_64 0:2.010-4.el7                  perl-Storable.x86_64 0:2.45-3.el7                           perl-Text-ParseWords.noarch 0:3.29-4.el7
  perl-Time-HiRes.x86_64 4:1.9725-3.el7             perl-Time-Local.noarch 0:1.2300-2.el7                       perl-constant.noarch 0:1.27-2.el7
  perl-libs.x86_64 4:5.16.3-294.el7_6               perl-macros.x86_64 4:5.16.3-294.el7_6                       perl-parent.noarch 1:0.225-244.el7
  perl-podlators.noarch 0:2.5.1-3.el7               perl-threads.x86_64 0:1.87-4.el7                            perl-threads-shared.x86_64 0:1.43-6.el7
  tcl.x86_64 1:8.5.13-8.el7

Replaced:
  mariadb-libs.x86_64 1:5.5.64-1.el7

Complete!
```

Installation loads software to the system. This software requires post-install actions and configuration before the database server is ready for use.

MariaDB ColumnStore post-installation scripts fails if they find MariaDB Enterprise Server running on the system. Stop the Server and disable the service after installing the packages:

```bash
$ sudo systemctl stop mariadb.service
$ sudo systemctl disable mariadb.service
```



### Install via Package Tarball

#TBD





## 4) Configuration

### MariaDB Server Configuration

MariaDB Enterprise Servers can be configured in the following ways:

+ System variables and options can be set in a configuration file (such as /etc/my.cnf). MariaDB Enterprise Server must be restarted to apply changes made to the configuration file.
+ System variables and options can be set on the command-line.
+ If a system variable supports dynamic changes, t hen it can be set on-the-fly using `SET` statement

For example, to configure MariaDB Enterprise Server via a configuration file:

```bash
# vi /etc/my.cnf.d/server.cnf
...
[mariadb]
datadir=/mariadb/data
log_error=/mariadb/log/mariadb-err.log
log_warnings=2
log_bin=/mariadb/log/mariadb-bin
...

```



### Storage Manager Configuration For S3

MariaDB ColumnStore can use any object store that is Amazon S3 API compatible. ColumnStore's "Storage Manager" uses a persistent disk cache for read/write operations so that it has minimal performance impact on ColumnStore. In some cases it will perform better than local disk operations.

If you want to utilize any sort of object store, you need to install and configure it prior to `postConfigure`.

Edit the `/etc/columnstore/storagemanager.cnf` file to configure Amazon S3 modify `[ObjectStorage]`, `[S3]` sections: 

```bash
[ObjectStorage]
service = S3

[S3]
region = <S3-Region>
bucket = <S3-Bucket-Name>
aws_access_key_id = <AWS-Access-Key-ID>
aws_secret_access_key = <AWS-Secret-Access-Key>
```



For example,

```bash
[ObjectStorage]
# 'service' is the module that SM will use for cloud IO.
# Current options are “LocalStorage” and “S3”.
# ‘LocalStorage’ will use a directory on the local filesystem as if it
# were cloud storage.  ‘S3’ is the module that uses real cloud storage.
# Both modules have their own sections below.
#
# Note, changing this after running postConfigure will leave you with an
# an inconsistent view of the data.
service = S3

...

[S3]
# These should be self-explanatory.  Region can be blank or commented
# if using a private cloud storage system.  Bucket has to be set to
# something though.  Obviously, do not change these after running
# postConfigure, or SM will not be able to find your data.
region = ap-northeast-2
bucket = kb-x4test-ap2

# Specify the endpoint to connect to if using an S3 compatible object
# store like Google Cloud Storage or IBM's Cleversafe.
# endpoint = storage.googleapis.com

# For the best performance do not specify a prefix.  It may be useful,
# however, if you must use a bucket with other data in it.  Because
# of the way object stores route data and requests, keep the
# prefix as short as possible for performance reasons.
# prefix = cs/

# Put your HMAC access keys here.  Keys can also be set through the
# environment vars AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY.
# If set, SM will use these values and ignore the envvars.
aws_access_key_id = <access-key-id>
aws_secret_access_key = <secret-access-key>
```





## 5) MariaDB Platform X4 Post Installation

### Post-Installation Script

Run the `columnstore-post-install` script on each ColumnStore Instance to provision the system to host the storage back-end:

```bash
$ sudo columnstore-post-install
```



### Post-Configuration Script

MariaDB ColumnStore provides a post-configuration script to configure the ColumnStore Instance. Post-configuration is only required on the first Server in your deployment (called pm1 by convention). The `postConfigure` script connects over SSH to initialize Performance Modules on the other Servers.

Run the `postConfigure` script on the Server hosting the initial Performance Module (that is, pm1):

```bash
$ sudo postConfigure
```

The `postConfigure` script runs through a series of prompts to configure the ColumnStore deployment.

+ When prompted to configure the ColumnStore deployment for single- or multi-node, select multi-node.
+ Sets the system name.
+ Configures the storage mount to store data internally on the local file system, externally on a network mounted disk, or using an S3 or GlusterFS storage manager.
+ Configures the Performance Modules, setting the IP addresses and DBRoots for each.
+ Sets the password to use in establishing SSH connections to the Performance Modules.

Once `postConfigure` has the information it needs, it connects to each Server and sets up the Performance Module to match the configuration, then starts MariaDB ColumnStore.



Installation Log:

```bash
[root@xdb1 ~]# postConfigure

This is the MariaDB ColumnStore System Configuration and Installation tool.
It will Configure the MariaDB ColumnStore System and will perform a Package
Installation of all of the Servers within the System that is being configured.

IMPORTANT: This tool requires to run on the Performance Module #1

Prompting instructions:

	Press 'enter' to accept a value in (), if available or
	Enter one of the options within [], if available, or
	Enter a new value


===== Setup System Server Type Configuration =====

There are 2 options when configuring the System Server Type: single and multi

  'single'  - Single-Server install is used when there will only be 1 server configured
              on the system. It can also be used for production systems, if the plan is
              to stay single-server.

  'multi'   - Multi-Server install is used when you want to configure multiple servers now or
              in the future. With Multi-Server install, you can still configure just 1 server
              now and add on addition servers/modules in the future.

Select the type of System Server install [1=single, 2=multi] (2) > 2


===== Setup System Module Type Configuration =====

There are 2 options when configuring the System Module Type: separate and combined

  'separate' - User and Performance functionality on separate servers.

  'combined' - User and Performance functionality on the same server

Select the type of System Module Install [1=separate, 2=combined] (1) > 2

Combined Server Installation will be performed.
The Server will be configured as a Performance Module.
All MariaDB ColumnStore Processes will run on the Performance Modules.

NOTE: The MariaDB ColumnStore Schema Sync feature will replicate all of the
      schemas and InnoDB tables across the User Module nodes. This feature can be enabled
      or disabled, for example, if you wish to configure your own replication post installation.

MariaDB ColumnStore Schema Sync feature, do you want to enable? [y,n] (y) > y


NOTE: MariaDB ColumnStore Replication Feature is enabled

Enter System Name (columnstore-1) > X4


===== Setup Storage Configuration =====


----- Setup Performance Module DBRoot Data Storage Mount Configuration -----

Columnstore supports the following storage options...
  1 - internal.  This uses the linux VFS to access files and does
      not manage the filesystem.
  2 - external *.  If you have other mountable filesystems you would
      like ColumnStore to use & manage, select this option.
  3 - GlusterFS *  Note: glusterd service must be running and enabled on
      all PMs.
  4 - S3-compatible cloud storage *.  Note: that should be configured
      before running postConfigure (see storagemanager.cnf)
  * - This option enables data replication and server failover in a
      multi-node configuration.

These options are available on this system: [1, 2, 4]
Select the type of data storage (1) > 4

===== Setup Memory Configuration =====


NOTE: Setting 'NumBlocksPct' to 50%
      Setting 'TotalUmMemory' to 25%


===== Setup the Module Configuration =====


----- Performance Module Configuration -----
# ssh_config(5) for more information.  This file provides defaults for

Enter number of Performance Modules [1,1024] (1) > 2

*** Parent OAM Module Performance Module #1 Configuration ***

Enter Nic Interface #1 Host Name (xdb1) >
Enter Nic Interface #1 IP Address or hostname of xdb1 (10.0.0.53) >
Enter Nic Interface #2 Host Name (unassigned) >
Enter the list (Nx,Ny,Nz) or range (Nx-Nz) of DBRoot IDs assigned to module 'pm1' (1) >

*** Performance Module #2 Configuration ***

Enter Nic Interface #1 Host Name (unassigned) > xdb2
Enter Nic Interface #1 IP Address or hostname of xdb2 (0.0.0.0) > 10.0.0.203
Enter Nic Interface #2 Host Name (unassigned) >
Enter the list (Nx,Ny,Nz) or range (Nx-Nz) of DBRoot IDs assigned to module 'pm2' () > 2

===== Running the MariaDB ColumnStore MariaDB Server setup scripts =====

post-mysqld-install Successfully Completed
post-mysql-install Successfully Completed

Next step is to enter the password to access the other Servers.
This is either user password or you can default to using an ssh key
If using a user password, the password needs to be the same on all Servers.

Enter password, hit 'enter' to default to using an ssh key, or 'exit' >
Confirm password >

----- Performing Install on 'pm2 / xdb2' -----

Install log file is located here: /tmp/columnstore_tmp_files/pm2_binary_install.log

===== Checking MariaDB ColumnStore System Logging Functionality =====

The MariaDB ColumnStore system logging is setup and working on local server

MariaDB ColumnStore System Configuration and Installation is Completed

===== MariaDB ColumnStore System Startup =====

System Configuration is complete.
Performing System Installation.

----- Starting MariaDB ColumnStore on local server -----

MariaDB ColumnStore successfully started

MariaDB ColumnStore Database Platform Starting, please wait ............... DONE

System Catalog Successfully Created

Run MariaDB ColumnStore Replication Setup..  DONE

MariaDB ColumnStore Install Successfully Completed, System is Active

Enter the following command to define MariaDB ColumnStore Alias Commands

. /etc/profile.d/columnstoreAlias.sh

Enter 'mariadb' to access the MariaDB ColumnStore SQL console
Enter 'mcsadmin' to access the MariaDB ColumnStore Admin console

NOTE: The MariaDB ColumnStore Alias Commands are in /etc/profile.d/columnstoreAlias.sh

```



### Restart the System

When the `postConfigure` script is complete, use mcsadmin to restart MariaDB ColumnStore to clear the cache:

```bash
$ sudo mdsadmin restartSystem
```



## 6) MariaDB Platform X4 Non-root Adjustment (Optional)

If you would like to use non-root user to manage MariaDB Platform X4, you need to apply the following adjustments on all nodes before proceeding `postConfigure` step.



### Stop Platform X4

First, stop the Platform X4 system. Execute the following commands on pm1 node to stop all ColumnsStore & MariaDB Enterprise Server processes:

```bash
# mcsadmin shutdownSystem y
```



### Non-root User Add

Create a new user. 

In this document, we are going to use `columnstore:columnstore` as a non-root user.

```bash
$ sudo groupadd columnstore
$ sudo useradd -g columnstore columnstore
$ sudo passwd columnstore
```



### Setup /etc/sudoers

Set up `/etc/sudoers` file so that the non-root user can manage Platform X4 system.

```bash
$ sudo echo "columnstore     ALL=(root)     NOPASSWD: /usr/bin/mcsadmin, /usr/bin/systemctl stop columnstore, /usr/bin/systemctl start columnstore, /usr/bin/systemctl stop mariadb, /usr/bin/systemctl start mariadb, /usr/bin/systemctl restart mariadb, /usr/bin/systemctl restart columnstore, /usr/bin/systemctl status columnstore, /usr/bin/systemctl status mariadb" >> /etc/sudoers
```



### ColumnStore Filesystem Ownership

You need to change the ownership of ColumnStore uses filesystem to be owned by non-root user.

For example:

```bash
$ sudo chown -R columnstore:columnstore /etc/columnstore/ /var/lib/columnstore/ /tmp/columnstore_tmp_files/ /var/log/mariadb

$ sudo chmod -R g+s /etc/columnstore/ /var/lib/columnstore/ /tmp/columnstore_tmp_files/ /var/log/mariadb
```



### Storage Manager FileSystem

You need to copy/move ColumnStore's "Storage Manager" uses a persistent disk cache Filesystem so non-root user can use it.

For example:

```bash
$ sudo cp -r /root/storagemanager /home/columnstore/
$ sudo chown -R columnstore:columnstore /home/columnstore/
```



### Storage Manager Configuration

There are paths you need to change for non-root user in a configuration file. Edit the `/etc/columnstore/storagemanager.cnf`, modifying the following paths to new Storage Manager Filesystem path you copied above.

```bash
[ObjectStorage]
metadata_path = ${HOME}/storagemanager/metadata
journal_path = ${HOME}/storagemanager/journal

[LocalStorage]
path = ${HOME}/storagemanager/fake-cloud

[Cache]
path = ${HOME}/storagemanager/cache
```



For example:

```bash
[ObjectStorage]
metadata_path = /home/columnstore/storagemanager/metadata
journal_path = /home/columnstore/storagemanager/journal

[LocalStorage]
path = /home/columnstore/storagemanager/fake-cloud

[Cache]
path = /home/columnstore/storagemanager/cache
```



## 7) Platform X4 Administration & Testing

### Start/Stop

MariaDB ColumnStore includes an administrative utility called `mcsadmin`, which you can use to start and stop the ColumnStore processes:

| Operation | User     | Command                                                      | On        |
| :-------- | -------- | :----------------------------------------------------------- | --------- |
| Start     | root     | `mcsadmin startSystem`                                       | pm1       |
| Start     | non-root | `sudo systemctl start columnstore && sudo systemctl start mariadb` | All Nodes |
| Stop      | All      | `mcsadmin stopSystem`                                        | pm1       |
| Stop      | non-root | `sudo systemctl stop columnstore && sudo systemctl stop mariadb` | All Nodes |



### Replication



### Cross Engine Joins



### Backup/Restore



### Scale-out/in





## 8) Enabling Smart Transaction (HTAP Replication)

Using MariaDB Replication, MariaDB Enterprise Server replicates writes from InnoDB tables to the MariaDB ColumnStore tables, ensuring that the application can perform analytical processing on current data. Combining MariaDB Replication with MariaDB MaxScale configured as a Binlog Server, MariaDB Enterprise Server can host InnoDB and ColumnStore on the same Server.



### Enable Replication for Smart Transactions

Replication must be enabled on a per-table basis to enable MariaDB Platform for Smart Transactions. To enable replication between an InnoDB and a ColumnStore table:

1. Create the respective InnoDB and ColumnStore tables. They must be in different databases and have the same names and structures. For example:

   ```mariadb
   CREATE DATABASE innodb_test;
   CREATE TABLE innodb_test.test (id INT) ENGINE = InnoDB;
   CREATE DATABASE columnstore_test;
   CREATE TABLE columnstore_test.test (id INT) ENGINE = ColumnStore;
   ```

2. Use the SET_HTAP_REPLICATION() UDF to enable replication between the above tables. For example:

   ```mariadb
   SELECT SET_HTAP_REPLICATION('<table_name_regex>','<source_database>', '<target_database>')
   ```

   Where: <table_name_regex> is a regex that matches all tables that need to be replicated;

   <source_database> is the name of the source database; <target_database> is the name of the target database.

   For example:

   ```mariadb
   SELECT SET_HTAP_REPLICATION('test', 'innodb_test', 'columnstore_test');
   ```

   

### Implemetation Guidance

To make working with HTAP replication easier:

+ Add a prefix and/or a suffix to the database and table names involved in the replication. This will make finding them with regex easier and less error prone.

+ Force a minimum database and/or table name length. This will reduce the chance of a false positive regex match

+ Never use table and database names that fully or partially match SQL keywords.

+ Avoid table and database names that include words that will be found in the data. Ensuring that there is no full or partial match to data will prevent problems.

+ Use the USE database syntax instead of database.table. This will make parsing much easier.

  

### Replication Management

#### List Replication

List the currently active replication with the SHOW_HTAP_REPLICATION() function:

```mariadb
SELECT SHOW_HTAP_REPLICATION();
```

The result will be something like this:

```mariadb
+------------------------------------------------------------------------------------+
| SHOW_HTAP_REPLICATION()                                                            |
+------------------------------------------------------------------------------------+
|
    === replication_filter ===
    table: test
    source database: innodb_test
    target database: columnstore_test
|
+------------------------------------------------------------------------------------+
```



#### Edit Replication

Edit replication with the SET_HTAP_REPLICATION() function. The new settings will override the old ones.

```mariadb
SELECT SET_HTAP_REPLICATION('<table_name_regex>', '<source_database>', '<target_database>');
```

Where: <table_name_regex> is a regex that matches all tables that need to be replicated;

<source_database> is the name of the source database; <target_database> is the name of the target database.

For example:

```mariadb
SELECT SET_HTAP_REPLICATION('test1|test2', 'innodb_test', 'columnstore_test');
```



#### Delete Replication

Replication can be deleted using the SET_HTAP_REPLICATION() function with an empty table name regex. This requires that there is only one target database and that the source database field also supports a regex. For example:

```mariadb
SELECT SET_HTAP_REPLICATION('', '', '');
```



## 9) MaxScale Setup

### MaxScale Installation

### MaxScale Configuration





## 10) References

+ https://mariadb.com/docs/deploy/deployment-methods/

+ https://mariadb.com/docs/deploy/multi-columnstore/

+ https://mariadb.com/docs/deploy/single-columnstore-centos7-col14/
+ https://mariadb.com/resources/blog/deploying-mariadb-platform-x4/
+ https://mariadb.com/docs/skysql/operations/htap-replication/#enable-replication-for-smart-transactions
+ https://mariadb.com/kb/en/sample-platform-x3-implementation-for-transactional-and-analytical-workloads/

