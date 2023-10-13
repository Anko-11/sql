# MYSQL读写分离的高可用主从复制集群

### 项目环境：

​	VMware虚拟机7台、CentOS7.9、mysql-5.7.38、keepalived-1.3.5、mysqlrouter-8.0.23、ansible-2.9.27

### 项目步骤：

#### 1.准备4台mysql服务器（1台master，2台slave，1台延迟备份slave3）、2台mysql路由服务器、1台备份管理服务器。配置好主机名并固定ip地址。

##### 1.1mysql服务器主机名设置为master、slave1、slave2、slave3。

tip1：mysql在创建二进制日志以及其他文件时会以主机名命名，在myslq服务启动后修改主机名可能导致mysql服务无法启动。重新登录或重启服务器刷新主机名。
防火墙可能导致服务器无法连接。

[root@localhost ~]# hostnamectl set-hostname **[hostname]**

##### 1.2固定ip地址，刷新网络服务，关闭firewalld。

tip2：在后续主从复制搭建时从服务器会指定主服务器ip以获取日志文件，使用dhcp会导致从服务器拿不到日志文件。

[root@localhost ~]# vim /etc/sysconfig/network-scripts/ifcfg-ens33

BOOTPROTO="none"
NAME="ens33"
DEVICE="ens33"
ONBOOT="yes"
IPADDR=   **[本机IP地址]**
PREFIX=24
GATEWAY=   **[网关地址]**
DNS1=114.114.114.114

[root@localhost ~]# service network restart

[root@localhost ~]# service firewalld stop

[root@localhost ~]# systemctl disable firewalld

#### 2.配置备份管理服务器，使用ansible定义主机清单，建立与master的双向免密通道。

##### 2.1安装ansible，使用ansible自动化运维工具，进入/etc/ansible/hosts配置主机清单

[root@ansible ~]# vim /etc/ansible/hosts

定义两个主机组[组名]把地址或主机名加进去

[db]
192.168.150.136 #master
192.168.150.137 #slave1
192.168.150.138 #slave2
192.168.150.139 #slave3
[dbslave]
192.168.150.137 #slave1
192.168.150.138 #slave2
192.168.150.139 #slave3

##### 2.2建立免密通道，生成公钥追加至其他服务器

在master上。

[root@master ~]# ssh-keygen
[root@master ~]# ssh-copy-id root@192.168.150.140

在ansible上。建立与所有mysql服务器的免密通道

[root@sc-master backup]# 
[root@ansible ~]# ssh-keygen
[root@ansible ~]# ssh-copy-id root@192.168.150.136
[root@ansible ~]# ssh-copy-id root@192.168.150.137
[root@ansible ~]# ssh-copy-id root@192.168.150.138
[root@ansible ~]# ssh-copy-id root@192.168.150.139

#### 3.导出主服务器数据并下发数据至从服务器。使用mysqldump与计划任务定期备份日志给备份管理服务器。

##### 3.1导出下发数据

[root@master ~] mysqldump -uroot -p$passwd  --all-databases --triggers --routines --events  >masterdata.sql
[root@master ~]# scp masterdata.sql root@192.168.150.140:/root
[root@ansible ~]# ansible dbslave -m copy -a 'src=/root/masterdata.sql dest=/root'

##### 3.2计划任务

[root@master ~]# crontab -l
30 2 * * * bash /backup/backup_masterdb.sh 
[root@master ~]# cat /backup/backup_masterdb.sh 
#!/bin/bash

mkdir -p /backup
mysqldump -uroot -p'$passwd'  --all-databases --triggers --routines --events  >/backup/$(date +%Y%m%d%H%M%S)masterdata.sql
scp /backup/$(date +%Y%m%d%H%M%S)masterdata.sql 192.168.150.140:/backup

#### 4.在mysql服务器上开启GTID功能、二进制日志功能，安装半同步插件，在slave上导入主服务器数据和副本配置。延迟备份slave3以slave1服务器为主服务器延迟备分数据。

##### 4.1在主服务器上安装半同步插件，开启gitd、二进制日志，创建用户test给slave使用。

mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

mysql> grant all on \*.*  to 'test'@'%' identified by '123456#';

tip3：半同步复制在主服务器上与从服务器所下载的插件是不一样的，具体操作可详见(https://dev.mysql.com/doc/refman/5.7/en/replication-semisync-installation.html)

[root@master ~]# vim /etc/my.cnf

[mysqld_safe]

[client]
socket=/data/mysql/mysql.sock

[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8
log_bin #开启二进制日志
server_id = 1 #默认为0，id为0时拒绝与其他服务器的连接，注意id不要重复。
rpl_semi_sync_master_enabled=1 #开启半同步复制
rpl_semi_sync_master_timeout=1000 #同步超时时间，单位毫秒
gtid-mode=ON #开启GTID复制
enforce-gtid-consistency=ON

[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

[root@master ~]# service mysqld restart

##### 4.2在从服务器上安装半同步插件，开启gitd、二进制日志。（不需要做其他服务器的master可不开启二进制日志）

tip4：#在实现3级以上同步时需要在中间级开启log_slave_updates=ON，否则由replication机制的SQL线程读取relay-log而执行的SQL语句并不会记录到bin-log，那么就无法实现上述的三级级联同步。

mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

[root@master ~]# vim /etc/my.cnf

[mysqld_safe]

[client]
socket=/data/mysql/mysql.sock

[mysqld]
socket=/data/mysql/mysql.sock
port = 3306
open_files_limit = 8192
innodb_buffer_pool_size = 512M
character-set-server=utf8
log_bin #开启二进制日志
log_slave_updates=ON 
server_id = [**从服务器序号**] #默认为0，id为0时拒绝与其他服务器的连接，注意id不要重复。
rpl_semi_sync_slave_enabled=1 #开启半同步复制
gtid-mode=ON #开启GTID复制
enforce-gtid-consistency=ON

[mysql]
auto-rehash
prompt=\u@\d \R:\m  mysql>

[root@master ~]# service mysqld restart

##### 4.3在slave上导入主服务器数据和副本配置。

[root@ansible ~]# ansible dbslave -m shell -a 'mysql -uroot -p$passwd </root/masterdata.sql'

● 在slave1和slave2上：

mysql>stop slave；
mysql>reset master；
mysql>reset slave；
mysql> CHANGE MASTER TO MASTER_HOST="192.168.150.136",
 MASTER_USER="test",
 MASTER_PASSWORD='123456#',
 MASTER_PORT=3306,
 MASTER_AUTO_POSITION=1;
mysql>start slave；

可以使用show slave status\G查看slave信息
Slave_IO_Running: Yes
Slave_SQL_Running: Yes

● 在slave3上：
mysql> CHANGE MASTER TO MASTER_HOST="192.168.150.137",
 MASTER_USER="test",
 MASTER_PASSWORD='123456#',
 MASTER_PORT=3306,
 MASTER_AUTO_POSITION=1;
 CHANGE MASTER TO MASTER_DELAY = 10;    #延迟备份

##### 5.在路由服务器上使用mysqlrouter以slave为只读、master为可写实现读写分离。

##### 5.1在mysqlrouter1和mysqlrouter2上下载mysqlrouter，配置/etc/mysqlrouter下的配置文件

下载地址https://dev.mysql.com/downloads/

[root@mysqlrouter1 ~]# yum install  https://downloads.mysql.com/archives/get/p/41/file/mysql-router-community-8.0.23-1.el7.x86_64.rpm

[root@mysqlrouter1 ~]# cat /etc/mysqlrouter/mysqlrouter.conf |egrep -v "^#"


[DEFAULT]
logging_folder = /var/log/mysqlrouter
runtime_folder = /var/run/mysqlrouter
config_folder = /etc/mysqlrouter

[logger]
level = INFO

[keepalive]
interval = 60

[routing:slaves]
bind_address = 0.0.0.0:7001
destinations = 192.168.150.137:3306,192.168.150.138:3306
mode = read-only
connect_timeout = 1

[routing:masters]
bind_address = 0.0.0.0:7002
destinations = 192.168.150.136:3306
mode = read-write
connect_timeout = 1
[root@mysqlrouter1 ~]#

启动mysqlrouter服务
[root@mysqlrouter1 ~]# service mysqlrouter start
Redirecting to /bin/systemctl start mysqlrouter.service

##### 5.2在master上创建2个测试账号，一个是读的，一个是写的

mysql>grant all on \*.*  to 'write'@'%' identified by '123456#';

mysql>grant select on \*.*  to 'read'@'%' identified by '123456#';

可以使用连接数据库的软件进行测试，也可以用其他服务器测试。在其他服务器上测试可以使用
mysql -h 192.168.150.141 -P 7001 -uread -p'123456#'   #IP地址是router服务器地址
mysql -h 192.168.150.141 -P 7002 -uwrite -p'123456#' 

tip5：权限是账号所有权限，而非mysqlrouter配置权限，所以在使用项目外机器访问集群时一定要匹配好端口和账号。
例：若使用read账号访问7002端口则进行不了写的操作，使用write账号访问7001端口则会导致向slave服务器上写入数据，向slave写入数据会导致主从复制数据不同步而断开连接。（修复办法参考末尾）

tip6：mysqlroute的默认规则对于read-write模式：将采用“首个可用”算法，对于read-only模式：将采用“轮询”算法。首个可用指先在第一个配置服务器上读写数据，若不能连接则向其他read-write模式服务器依次请求连接；轮询指依次轮流向每个服务器读取数据。

#### 6.使用keepalived配置2个vrrp实例,实现双vip的高可用功能。

##### 6.1 在所有路由服务器上安装keepalived，配置keepalived.conf文件，启动服务

[root@mysqlrouter1 ~]# yum install keepalived 

[root@mysqlrouter1 ~]# vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
  #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 80
    priority 200 #优先级0~255
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.150.188
    }
}

[root@mysqlrouter1 ~]#service keepalived start

[root@mysqlrouter2 ~]# yum install keepalived 

[root@mysqlrouter2 ~]# vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
  #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {
    state backup
    interface ens33
    virtual_router_id 80
    priority 100 #优先级
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.150.188
    }
}

[root@mysqlrouter2 ~]# service keepalived start



#### EX：GTID半同步主从复制同步断开修复办法

首先在从服务器上查看GTIDS编号。

mysql>show slave status\G;

           Retrieved_Gtid_Set: ee952679-4323-11ee-8635-000c29247516:15-20
            Executed_Gtid_Set: ee952679-4323-11ee-8635-000c29247516:1-20

Retrieved_Gtid_Set：从库已经接收到主库的事务编号（从库的IO线程已经接受到了）

Executed_Gtid_Set：已经执行的事务编号（从库的执行sql线程已经执行了的sql）

##### 1.若slave上的GTIDs编号比master上的大

需要从新从主服务器上导出数据导入至从服务器，注意导入时不要开启GTID功能，导入后在开启。

[root@master ~]# mysqldump -uroot -p$passwd  --all-databases --triggers --routines --events  >masterdata.sql

在/etc/my.cnf文件里关闭GTID功能，重新启动mysqld进程。
修改这两行↓
gtid-mode=OFF
enforce-gtid-consistency=OFF
[root@slave ~]# service mysqld restart

导入新的二进制文件
[root@slave ~]# mysql -uroot -p$passwd </root/masterdata.sql

在/etc/my.cnf文件里开启GTID功能，重新启动mysqld进程。
修改这两行↓
gtid-mode=ON
enforce-gtid-consistency=ON
[root@slave ~]# service mysqld restart

##### 2.若slave上的GTIDs编号比master上的小

​	可以使用上述方法重新导入数据，也可以在数据一致的前提下手动修改GTID位置。此方法为热修复，可不需要餐厅mysql服务。

修改GITD位置方法：
mysql>stop slave；
mysql>reset slave；
mysql>set global gtid_purged=' ee952679-4323-11ee-8635-000c29247516:1-15';  #设置guid位置
CHANGE MASTER TO MASTER_HOST="192.168.150.136",
 MASTER_USER="test",
 MASTER_PASSWORD='123456#',
 MASTER_PORT=3306,
 MASTER_AUTO_POSITION=1;
mysql>start slave；

