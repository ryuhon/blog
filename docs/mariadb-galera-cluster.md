---
layout: default
title: MariaDB Galera Cluster 설치
nav_order: 3
---

# MariaDB Galera Cluster 환경 구축 

## 설치전 체크사항 
* 우분투 18.04 기준 
* mariadb 10.4 사용 

## mariadb 10.4 설치 
```bash
sudo apt-get install software-properties-common
``` 
```bash
sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
``` 
```bash
sudo add-apt-repository "deb [arch=amd64,arm64,ppc64el] http://mariadb.mirror.liquidtelecom.com/repo/10.4/ubuntu $(lsb_release -cs) main"
``` 
```bash
sudo apt update
sudo apt -y install mariadb-server mariadb-client
``` 
## 보안 설정 인스톨 
```bash
$ sudo mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] y
Enabled successfully!
Reloading privilege tables..
 ... Success!


You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```
## mysql 접속 테스트 
```bash
sudo mysql
```
## mariadb backup 설치 
```bash
sudo apt-get install mariadb-backup
```

## 갈레라 환경설정 전 체크 사항 
1. ### wsrep_sst_method 로 무엇을 쓸것인가? 
      * rsync 
      * mariabackup
1. ### 노드는 몇대로 할것인가?  
      * 최소 3대 이상 

## DATA 폴더 변경 (필요한 경우만)
```bash
systemctl stop mysql
rsync -av /var/lib/mysql /data/
```
```bash
# vi /etc/mysql/my.cnf
[mysqld]
datadir=/data/mysql
```
## utf8mb4 설정
```bash

# vi /etc/mysql/my.cnf

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

## 갈레라 연동 계정 설정 (각 노드별로 전부)
```bash
create user 'replication'@'node_ip' identified by 'password' ;
grant all privileges on *.* to replication@node_ip;
flush all privileges;
```

## 갈레라 환경설정
```bash
# vi /etc/mysql/my.cnf

[galera]
# Mandatory settings
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so #libgalera_smm.so의 위치 
wsrep_cluster_address="gcomm://192.168.0.1,192.168.0.2,192.168.0.3"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
wsrep_cluster_name='mariadb-cluster'      #클러스터명
wsrep_node_address='192.168.0.1'          #해당 노드 ip
wsrep_node_name='DEVBOX02'                #해당 노드 호스트네임
wsrep_sst_method=mariabackup              #동기화 방식. 10.4 이후라서 mariabackup 선택 
wsrep_sst_auth=replication:password       #상단에 설정한 갈레라 연동 계정
wsrep_sst_receive_address=192.168.0.1     #해당 노드 ip
wsrep_provider_options="gcache.recover=yes" 
#갈레라 클러스터중 노드가 종료되었을때 깨진 클러스터를 재구성하지 않고 DB를 되살리기 위한 옵션

```

## isolation 변경 (데드락 방지)
```bash
# vi /etc/mysql/my.cnf
[mysqld]
transaction-isolation           = READ-COMMITTED
```

## 구동
### 첫번째 노드 (마스터) 구동
```bash
galera_new_cluster
```
### 나머지 노드 구동
```bash
systemctl start mysqld
```
## 설치 및 구동 확인 
```bash
# mysql 진입후 
SHOW STATUS LIKE 'wsrep%';
```