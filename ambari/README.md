# EDP deployment

We adop Apache ambari and plus our enhancement on it to deploy our esse data platform.

## System Requirements

* Operating Systems Requirements
  - Red Hat Enterprise Linux (RHEL) v7.x
  - Red Hat Enterprise Linux (RHEL) v6.6
  - CentOS v7.x
  - CentOS v6.6

* Browser Requirements
  - Internet Explorer 9.0 (deprecated)
  - Firefox 18
  - Google Chrome 26
  - Safari 5

* JDK Requirements
  - Oracle JDK 1.8 64-bit (minimum JDK 1.8_40) (default)

* Database Requirements
  - PostgreSQL 8
  - PostgreSQL 9.1.13+,9.3
  - MySQL 5.6
  - Oracle 11gr2, 12c

* Memory Requirements
The Ambari server host should have at least 1 GB RAM, with 500 MB free.

To check available memory on any host, run:
```
free -m
```

| Number of hosts | Memory Available | Disk Space |
|-----------------|------------------|------------|
| 1  | 1024 MB | 10 GB
| 10 | 1024 MB | 20 GB
| 50 | 2048 MB | 50 GB
| 100| 4096 MB |100 GB
| 300| 4096 MB |100 GB
| 500| 8096 MB |200 GB
|1000|12288 MB |200 GB
|2000|16384 MB |500 GB

## Prepare before your deployment

First you need setup the docker env before run the following.

* Start Postgresql container
Create two databases for ambari server, hive and oozie. By default, Ambari will install an instance of MySQL for the Hive Metastore and Derby for Oozie Server, but for production environment, we strongly recommended to use existing Postgresql Database with High Availability and Replication enabled.
```
$ docker run -d --restart=always \
-v /data/postgresql:/var/lib/postgresql \
-p 5432:5432 \
-e DB_NAME=hive_meta,oozie,ambari \
-e DB_USER=dbuser \
-e DB_PASS=<db password> \
--name postgresql sameersbn/postgresql:9.4-2
```

*Note: you can run the following command to clean the database when try to reset the Ambari server.*
```
$ docker run -d --rm=true \
--entrypoint=bash \
-e PGPASSWORD=<db password> ambari:0.0.1 
psql -d ambari -h <db host> \
-U <db user> \
-f /var/lib/ambari-server/resources/Ambari-DDL-Postgres-DROP.sql \
-v username=<db user>
```

* Set up ambari server
  - Build docker image
```
$ cd docker/docker-ambari/ambari-server
$ docker build -t ambari:0.0.1 --force-rm=true ./
```

  - Start ambari server
```
$ docker run -d --restart=always -p 8080:8080 -p 8440:8440 -p 8441:8441 --name ambari-server -h ambari-server.esse.io -e POSTGRES_SERVER=<psq server> -e POSTGRES_PORT=5432 -e POSTGRES_DB=ambari -e POSTGRES_USER=<psq dbuser> -e POSTGRES_PWS=<psq password> --add-host='edp04.esse.io:192.168.250.15' --add-host='edp05.esse.io:192.168.250.16' ambari:0.0.1
```
*Note: If you have DNS in your deployment environment, you need not specify the --add-host when run the docker container, otherwise you need provide all  the pair of FQDN and IP of your Hadoop cluster nodes when run the docker container via --add-host.*

* Enable NTP on cluster
The clocks of all the nodes in your cluster must be able to synchronize with each other.

* Install JDK on cluster
Install the JDK8 on all the nodes in your cluster.
```
$ mkdir -p /usr/jdk64 && cd /usr/jdk64 && \
wget http://repo.esse.io:33080/ARTIFACTS/jdk-8u40-linux-x64.tar.gz && \
tar -xf jdk-8u40-linux-x64.tar.gz && \
rm -f jdk-8u40-linux-x64.tar.gz
```

* Check DNS and NSCD
All hosts in your system must be configured for both forward and and reverse DNS.

If you are unable to configure DNS in this way, you should edit the /etc/hosts file on every host in your cluster to contain the IP address and Fully Qualified Domain Name of each of your hosts.

Name Service Caching Daemon (NSCD), this daemon will cache host, user, and group lookups and provide better resolution performance, and reduced load on DNS infrastructure.

## Deploy Hadoop Cluster

* Hadoop repository
By default, we use the repository from [esse repository](http://repo.esse.io:33080/repos).

If you have no internet access in the cluster, you have to set up http mirror server on the Ambari server container host and create repository by your self.
  - Create HDP.repo under the /etc/yum.repos.d on your http server host for example:
```
[HDP-2.3]
name=HDP-2.3
baseurl=http://repo.esse.io:33080/repos/HDP/centos7/2.x/updates/2.3.2.0

path=/
enabled=1
```

  - Create HDP-UTILS.repo under the /etc/yum.repos.d on your http mirror server host for example:
```
[HDP-UTILS-1.1.0.20]
name=HDP-UTILS-1.1.0.20
baseurl=http://repo.esse.io:33080/repos/HDP-UTILS-1.1.0.20/repos/centos7

path=/
enabled=1
```

  - Create mysql-community.repo under the /etc/yum.repos.d on your http server host for example:
```
[mysql-connectors-community]
name=MySQL Connectors Community
baseurl=http://repo.esse.io:33080/repos/mysql-connectors-community/el/7/$basearch/
enabled=1
# Enable to use MySQL 5.6
[mysql56-community]
name=MySQL 5.6 Community Server
baseurl=http://repo.esse.io:33080/repos/mysql-5.6-community/el/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:/etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
```

  - Run *yum repolist* to confirm availability of the repositories.
```
!HDP-2.3                           HDP-2.3                         175
!HDP-UTILS-1.1.0.20                HDP-UTILS-1.1.0.20               43
!Updates-ambari-2.1.2              ambari-2.1.2 - Updates            6
!mysql-connectors-community/x86_64 MySQL Connectors Community       17
!mysql56-community/x86_64          MySQL 5.6 Community Server      184
```

  - Run *reposync HDP-2.3* to synchronize the repository contents to your mirror server for example.
  - Run *creatrepo <repo location>* to generate the repository metadata.
  - Create new mysql-community-release rpm if you want install new instance of MySQL for Hive or Oozie.
*Update the repo/el7/etc/yum.repos.d/mysql-community.repo to specify your internal http server host before create rpm package.

You can find the command line to create rpm package in the [repo/README.md](../repo/README.md)*

*Replace the existing mysql-community-release rpm under the HDP-UTILS-1.1.0.20/repos/centos7/mysql/ with the rpm package you created.*
