### 1 VMware16虚拟机安装配置

#### 1.1  网络配置

* 编辑-虚拟网络编辑器

选中虚拟机在用的虚拟网卡，修改 **子网IP** 和 **子网掩码**

* 点击 DHCP设置

修改 起始IP地址 和 结束IP地址

* hostname设置

hostname：centos7.lzy.org

#### 1.2 虚拟机磁盘分区

* 分区为标准分区 (Standard partition)

* / ：100G
* /boot：1G             文件类型：ext4
* swap：2G（与虚拟内存相等）
* 其余为数据挂载点，自行设置

#### 1.3 用户名/密码

* root/1234$#@!

* lzy/1234$#@!
* IP信息
  * centos7-test2：10.0.0.100
  * centos7-test3：10.0.0.101

### 2 Mysql部署

#### 2.1 navicat远程连接排查

* 确认mysql数据库服务是否起来

```bash
[root@centos7 script]# netstat -ntlp |grep 3306
tcp6       0      0 :::3306                 :::*                    LISTEN      1253/mysqld
[root@centos7 script]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-11-11 14:42:03 CST; 36min ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 1250 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 1232 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 1253 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─1253 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

Nov 11 14:42:02 centos7.lzy.org systemd[1]: Starting MySQL Server...
Nov 11 14:42:03 centos7.lzy.org systemd[1]: Started MySQL Server.
```

* 修改mysql数据库权限

```sql
use mysql;
select user,host from user;
update user set host = '%' where user = 'root';
--------限制某一网段的设备允许远程访问数据库--------
update user set host = '**.**.**.%' where user = 'root';
--------刷新---------
flush privileges;
```

* 查看防火墙端口是否开放

```bash
[root@centos7 script]# firewall-cmd --list-all
# 设置需要开放的端口
[root@centos7 script]# firewall-cmd --zone=public --add-port=3306/tcp --permanent
Warning: ALREADY_ENABLED: 3306:tcp
success
# 重启防火墙并查看是否生效
[root@centos7 script]# firewall-cmd --reload
success
[root@centos7 script]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens33
  sources: 
  services: ssh dhcpv6-client
  ports: 3306/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

#### 2.2 mysql二进制部署

##### 2.2.1 客户端程序命令

```http
mysql: 交互式或非交互式的CLI工具
mysqldump：备份工具，基于mysql协议向mysqld发起查询请求，并将查得的所有数据转换成
insert等写操作语句保存文本文件中
mysqladmin：基于mysql协议管理mysqld
mysqlimport：数据导入工具
MyISAM 存储引擎的管理工具：
myisamchk：检查MyISAM库
myisampack：打包MyISAM表，只读
```

##### 2.2.2 离线部署MySQL-5.6.*脚本

```bash
#!/bin/bash
DIR=`pwd`
MYSQL_VERSION=5.6.47
NAME="mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64.tar.gz"
FULL_NAME=${DIR}/${NAME}
DATE_DIR=/data/mysql

yum install -y libaio perl-Data-Dumper

if [ -f ${FULL_NAME} ];then
    echo "安装文件存在"
else
    echo "安装文件不存在"
fi

if [ -h /usr/local/mysql ];then
    echo "mysql已安装"
    exit 3
else
    tar xvf ${FULL_NAME} -C /usr/local/src
    ln -sv /usr/local/src/mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64 /usr/local/mysql
    if id mysql;then
        echo "mysql用户已存在，跳过创建用户的过程"
    else
        useradd -r -s /sbin/nologin mysql
    fi

    if id mysql;then
        chown -R mysql.mysql /usr/local/mysql/*
        if [ ! -d /data/mysql ];then
            mkdir -pv /data/mysql && chown -R mysql.mysql /data -R
            /usr/local/mysql/scripts/mysql_install_db --user=mysql --datadir=/data/mysql --basedir=/usr/local/mysql
            cp /usr/local/src/mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64/support-files/mysql.server /etc/init.d/mysqld
            chmod a+x /etc/init.d/mysqld
cat > /etc/my.cnf <<EOF
[mysqld]
socket=/data/mysql/mysql.sock
user=mysql
symbolic-links=0
datadir=/data/mysql
innodb_file_per_table=1

[client]
port=3306
socket=/data/mysql/mysql.sock

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/tmp/mysql.sock
EOF

            ln -sv /usr/local/mysql/bin/mysql /usr/bin/mysql
            /etc/init.d/mysqld start
            chkconfig --add mysqld
        else
            echo "Mysql 数据目录已存在"
            exit 3
        fi
    fi
fi
```

##### 2.2.3 离线部署MySQL-5.7.*脚本

```bash
#!/bin/bash
. /etc/init.d/functions
DIR=`pwd`
MYSQL_VERSION=5.7.39
MYSQL="mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64.tar.gz"
COLOR="echo -e \E[01,31m"
END="\E[0m"
MYSQL_ROOT_PASSWORD="123456"

check (){

if [ $UID -ne 0 ];then
    action "user is not root,install failed..." false
    exit 1
fi

cd $DIR
if [ ! -e $MYSQL ];then
    $COLOR"lack ${MYSQL} files..."$END
    $COLOR"Please put the software in the ${DIR} directory..."$END
    exit
elif [ -e /usr/local/mysql ];then
    action "MySQL alrady exists,install faild..." false
    exit
else
    return
fi
}

install_mysql (){
    $COLOR"Start installing MySQL databases..."$END
    yum  -y -q install libaio numactl-libs 
    cd $DIR
    tar xvf ${MYSQL} -C /usr/local
    MYSQL_DIR=`echo $MYSQL| sed -nr 's/^(.*[0-9]).*/\1/p'`
    ln -s /usr/local/${MYSQL_DIR} /usr/local/mysql
    chown -R root.root /usr/local/mysql
    id mysql &> /dev/null || { useradd -s /sbin/nologin -r mysql ; action "create mysql user..."; }
    echo 'PATH=/usr/local/mysql/bin/:$PATH' > /etc/profile.d/mysql.sh
    . /etc/profile.d/mysql.sh
    ln -s /usr/local/mysql/bin/* /usr/bin
    cat > /etc/my.cnf <<-EOF
[mysqld]
server-id=1
log-bin
datadir=/data/mysql
socket=/data/mysql/mysql.sock                                                   
                                                
log-error=/data/mysql/mysql.log
pid-file=/data/mysql/mysql.pid
[client]
socket=/data/mysql/mysql.sock
EOF
    [ -d /data ] || mkdir /data
    mysqld --initialize --user=mysql --datadir=/data/mysql
    cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
    chkconfig --add mysqld
    chkconfig mysql on
    service mysqld start
    [ $? -ne 0] && { $COLOR"Databases start failed,exit..."$END;exit; }
    sleep 3
    MYSQL_OLDPASSWORD=`awk '/A temporary password/{print $NF}' /data/mysql/mysql.log`
    mysqladmin -uroot -p$MYSQL_OLDPASSWORD password $MYSQL_ROOT_PASSWORD &> /dev/null
    action "Database install complete..."
}

check
install_mysql
```

#### 2.3 shell 

##### 2.3.1 shell表达式

```ABAP
#文件表达式
-e filename 如果 filename存在，则为真
-d filename 如果 filename为目录，则为真 
-f filename 如果 filename为常规文件，则为真
-L filename 如果 filename为符号链接，则为真
-r filename 如果 filename可读，则为真 
-w filename 如果 filename可写，则为真 
-x filename 如果 filename可执行，则为真
-s filename 如果文件长度不为0，则为真
-h filename 如果文件是软链接，则为真
filename1 -nt filename2 如果 filename1比 filename2新，则为真
filename1 -ot filename2 如果 filename1比 filename2旧，则为真

#整数变量表达式
-eq 等于
-ne 不等于
-gt 大于
-ge 大于等于
-lt 小于
-le 小于等于

# 字符串变量表达式
If  [ $a = $b ]                 如果string1等于string2，则为真; 字符串允许使用赋值号做等号
if  [ $string1 !=  $string2 ]   如果string1不等于string2，则为真       
if  [ -n $string  ]             如果string 非空(非0），返回0(true)  
if  [ -z $string  ]             如果string 为空，则为真
if  [ $sting ]                  如果string 非空，返回0 (和-n类似) 

# 逻辑表达式
逻辑非 !                  		条件表达式的相反
if [ ! 表达式 ]
if [ ! -d $num ]               如果不存在目录$num

逻辑与 –a                       条件表达式的并列
if [ 表达式1  –a  表达式2 ]

逻辑或 -o                       条件表达式的或
if [ 表达式1  –o 表达式2 ]
```

##### 2.3.2 shell语法

```bash
# for 循环
for 变量名 in 取值列表
do
  命令
done
```

#### 2.4 mysql备份&恢复

##### 2.4.1 mysql日志

```bash
############# 事务日志：transaction log
# redo log:实现 WAL（Write Ahead Log) ,数据更新前先记录redo log
# undo log：保存与执行的操作相反的操作,用于实现rollback
mysql> show variables like 'innodb_log%';
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| innodb_log_buffer_size      | 16777216 |
| innodb_log_checksums        | ON       |
| innodb_log_compressed_pages | ON       |
| innodb_log_file_size        | 50331648 |			# 每个日志文件大小
| innodb_log_files_in_group   | 2        |			# 日志组成员个数
| innodb_log_group_home_dir   | ./       |			# 事务文件路径
| innodb_log_write_ahead_size | 8192     |
+-----------------------------+----------+
7 rows in set (0.00 sec)
# 事务日志性能调优
innodb_flush_log_at_trx_commit=0 or 1 or 2
1 此为默认值，日志缓冲区将写入日志文件，并在每次事务后执行刷新到磁盘。 这是完全遵守ACID特性
0 提交时没有写磁盘的操作; 而是每秒执行一次将日志缓冲区的提交的事务写入刷新到磁盘。 这样可提供更好的性能，但服务器崩溃可能丢失最后一秒的事务
2 每次提交后都会写入OS的缓冲区，但每秒才会进行一次刷新到磁盘文件中。 性能比0略差一些，但操作系统或停电可能导致最后一秒的交易丢失
```

```bash
############# 错误日志：
# mysqld启动和关闭过程中输出的事件信息
# mysqld运行中产生的错误信息
# event scheduler运行一个event时产生的日志信息
# 在主从复制架构中的从服务器上启动从服务器线程时产生的信息
mysql> show global variables like '%error%';
+---------------------+-----------------------+
| Variable_name       | Value                 |
+---------------------+-----------------------+
| binlog_error_action | ABORT_SERVER          |
| log_error           | /data/mysql/mysql.log |			# 错误日志存储路径
| log_error_verbosity | 3                     |
| max_connect_errors  | 100                   |
| max_error_count     | 64                    |
| slave_skip_errors   | OFF                   |
+---------------------+-----------------------+
6 rows in set (0.00 sec)
```

```bash
############# 通用日志：记录数据库的操作过程
select @@general_log;					#查询通用日志是否开启
set global general_log = 1;				 #开启通用日志
show global variables like "log_output";  #默认日志存储在文件中
show variables like "%general%";		 #查看通用日志文件保存路径

show table status like 'general_log'\G;   # \G参数序列化展示数据！！！
select argument,count(argument) num from mysql.general_log group by argument order by num desc limit 3;
select command_type,count(command_type) num from mysql.general_log group by command_type order by num desc limit 4;
# 在通用日志表中查询排名前三的操作（不同维度统计）
mysql -e 'select argument from mysql.general_log' | awk '{sql[$0]++}END{for(i in sql){print sql[i],i}}'| sort -nr
mysql -e 'select argument from mysql.general_log' | sort | uniq -c | sort -nr
# 在通用日志文件中对访问语句进行排序
```

```bash
############# 慢查询日志：记录执行查询时长超出指定时长的操作
slow_query_log=ON|OFF             #开启或关闭慢查询，支持全局和会话，只有全局设置才会生成慢查询文件
long_query_time=N        		 #慢查询的阀值，单位秒,默认为10s
slow_query_log_file=HOSTNAME-slow.log  #慢查询日志文件
log_slow_filter = admin,filesort,filesort_on_disk,full_join,full_scan,
query_cache,query_cache_miss,tmp_table,tmp_table_on_disk 
#上述查询类型且查询时长超过long_query_time，则记录日志
log_queries_not_using_indexes=ON  #不使用索引或使用全索引扫描，不论是否达到慢查询阀值的语句是否记录日志，默认OFF，即不记录
log_slow_rate_limit = 1           #多少次查询才记录，mariadb特有
log_slow_verbosity= Query_plan,explain #记录内容
log_slow_queries = OFF            #同slow_query_log，MariaDB 10.0/MySQL 5.6.1 版后已删除

show variables like 'slow%';
show variables like '%quer%';
select sleep(20);						#函数执行慢查询操作，验证日志是否记录

create index idx_age on students(age);	  #在表中某个字段创建索引
explain select * from students where age = 20;		#查询该命令执行中是否使用了索引
```

```bash
############# 二进制日志：记录（潜在）导致数据改变的sql | 记录已提交的日志 | 不依赖存储引擎类型
# 建议二进制日志文件与数据文件分开存放
# 二进制日志记录的三种格式
mysql> show variables like 'binlog_format%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
.基于"语句"记录：statement，记录语句，默认模式，日志量较少;数据不全
.基于"行"记录：row，记录数据，日志量较大，更加安全，建议使用的格式,MySQL8.0默认格式
.混合模式：mixed, 让系统自行判定该基于哪种方式进行

# 二进制日志文件构成
.日志文件：mysql/mariadb-bin.文件名后缀，二进制格式
.索引文件：mysql/mariadb-bin.index，文本格式,记录当前已有的二进制日志文件列表

# 二进制日志相关变量
sql_log_bin=ON|OFF：			#是否记录二进制日志，默认ON，支持动态修改，系统变量，而非服务器选项（临时选项）
log_bin=/PATH/BIN_LOG_FILE：  #指定文件位置；默认OFF，表示不启用二进制日志功能，上述两项都开启才可以
mysql> show variables like '%log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
| sql_log_bin   | ON    |
+---------------+-------+
[root@centos7 mysql_logbin]# more /etc/my.cnf
[mysqld]
******
log-bin=/data/mysql_logbin/logbin	# log-bin配置文件，最后表示二进制日志文件的前缀
******

max_binlog_size=1073741824：	#单个二进制日志文件的最大体积，到达最大值会自动滚动，默认为1G
#说明：文件达到上限时的大小未必为指定的精确值
binlog_cache_size=4m 		#此变量确定在每次事务中保存二进制日志更改记录的缓存的大小（每次连接）
max_binlog_cache_size=512m 	 #限制用于缓存多事务查询的字节大小。
sync_binlog=1|0：		   #设定是否启动二进制日志即时同步磁盘功能，默认0，由操作系统负责同步日志到磁盘
expire_logs_days=N：		    #二进制日志可以自动删除的天数。 默认为0，即不自动删除

# 查看所有的二进制日志
show master/binary logs;
# 查看当前正在使用的二进制日志
show master status；
# 查看二进制日志中的内容
show binlog events;
show binlog events in 'Log_name';
SHOW BINLOG EVENTS in 'mysql-bin.000002' from 614 limit 2,3;\G

# 查看二进制日志文件的工具：mysqlbinlog
mysqlbinlog [OPTIONS] log_file…
 --start-position=	# 指定开始位置
 --stop-position=
 --start-datetime=  #时间格式：YYYY-MM-DD hh:mm:ss
 --stop-datetime= 
 --base64-output[=name]
        -v -vvv
        
################二进制日志文件格式#################
事件发生的日期和时间：151105 16:31:40
事件发生的服务器标识：server id 1
事件的结束位置：end_log_pos 431
事件的类型：Query 
事件发生时所在服务器执行此事件的线程的ID：thread_id=1
语句的时间戳与将其写入二进制文件中的时间差：exec_time=0
错误代码：error_code=0
事件内容：
GTID：Global Transaction ID，mysql5.6以mariadb10以上版本专属属性：GTID
#################################################

# 清除二进制日志
mysql> purge binary/master logs {to 'log_name'}/{before datetime_expr}
PURGE BINARY LOGS TO 'mariadb-bin.000003'; 		# 删除mariadb-bin.000003之前的日志
PURGE BINARY LOGS BEFORE '2017-01-23';		    # 删除2017-01-23日之前的日志
PURGE BINARY LOGS BEFORE '2017-03-22 09:25:30';
mysql> reset master;						  # 全部清除二进制日志，从000001开始计数


# 重新生成二进制文件/切换日志文件
mysql> flush logs;
[root@centos7 mysql_logbin]# mysqladmin -uroot -p123456 flush-logs
```

##### 2.4.2 数据备份&恢复

```bash
# 备份类型1
增量备份：仅备份最近一次完全备份或增量备份（如果存在增量）以来变化的数据，备份较快，还原复杂
差异备份：仅备份最近一次完全备份以来变化的数据，备份较慢，还原简单
# 备份类型2
冷备：读、写操作均不可进行，数据库停止服务
温备：读操作可执行；但写操作不可执行
热备：读、写操作均可执行
  MyISAM：温备，不支持热备
  InnoDB：都支持
# 备份类型3
物理备份：直接复制数据文件进行备份，与存储引擎有关，占用较多的空间，速度快
逻辑备份：从数据库中"导出"数据另存而进行的备份，与存储引擎无关，占用空间少，速度慢，可能丢失精度
```

```bash
# 冷备
1.数据库服务停止
2.cp/tar 数据库目录至本地/异地       #常规的/var/lib/mysql/
3.二进制安装/etc/my.cnf,也需要备份
4.重装
5.恢复备份数据（cp回原位置 cp -a保留属性）
```

```bash
# mysqldump 备份工具
[root@centos7 mysql_logbin]# mysqldump
Usage: mysqldump [OPTIONS] database [tables]
OR     mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
OR     mysqldump [OPTIONS] --all-databases [OPTIONS]

-A, --all-databases 		#备份所有数据库，含create database
-B, --databases db_name…  	#指定备份的数据库，包括create database语句
-E, --events：			   #备份相关的所有event scheduler
-R, --routines：			   #备份所有存储过程和自定义函数
--triggers：				   #备份表相关触发器，默认启用,用--skip-triggers，不备份触发器
--default-character-set=utf8 #指定字符集
--master-data[=#]： 		    #此选项须启用二进制日志
#1：所备份的数据之前加一条记录为CHANGE MASTER TO语句，非注释，不指定#，默认为1，适合于主从复制多机使用
#2：记录为被注释的#CHANGE MASTER TO语句，适合于单机使用,适用于备份还原
#此选项会自动关闭--lock-tables功能，自动打开-x | --lock-all-tables功能（除非开启--single-transaction）
-F, --flush-logs 			#备份前滚动日志，锁定表完成后，执行flush logs命令,生成新的二进制日志文件，
# 配合-A 或 -B 选项时，会导致刷新多次数据库。建议在同一时刻执行转储和日志刷新，可通过和--singletransaction或-x，--master-data 一起使用实现，此时只刷新一次二进制日志
--compact        			#去掉注释，适合调试，节约备份占用的空间,生产不使用
-d, --no-data    			#只备份表结构,不备份数据,即只备份create table 
-t, --no-create-info 		#只备份数据,不备份表结构,即不备份create table 
-n, --no-create-db 			#不备份create database，可被-A或-B覆盖
--flush-privileges 			#备份mysql或相关时需要使用
-f, --force         		#忽略SQL错误，继续执行
--hex-blob       			#使用十六进制符号转储二进制列，当有包括BINARY， VARBINARY，
BLOB，BIT的数据类型的列时使用，避免乱码
-q, --quick    				#不缓存查询，直接输出，加快备份速度

#命令
[root@centos7 ~]# mysqldump -uroot -p -A --master-data=1 > /root/all.sql
#数据库损坏
[root@centos7 ~]# grep '^CHANGE MASTER TO' /root/all.sql 
CHANGE MASTER TO MASTER_LOG_FILE='logbin.000001', MASTER_LOG_POS=10792;
[root@centos7 ~]# mysqlbinlog --start-position=10792 /data/mysql_logbin/logbin.000001 > /root/binlog.sql
#修复数据库
mysql> set sql_log_bin = 0;					# 临时关闭数据库二进制日志
mysql> source /root/all.sql					# 按序先恢复全量备份的数据
mysql> source /root/binlog.sql				# 再恢复增量备份的数据
mysql> set sql_log_bin = 1;					# 开启二进制日志
```

##### 2.4.3 备份脚本

```bash
# 备份脚本
#!/bin/bash
PASSWD=123456
DIR=/data/backup/`date +%F`
DEL_DIR=
All_DB=`mysql -uroot -p${PASSWD} -e 'show databases' |grep -Ev 'Database|information_schema|performance_schema|sys'`

[ -d "$DIR" ] || mkdir -pv $DIR
for db in ${All_DB}
do
    mysqldump -uroot -p${PASSWD} -F --single-transaction --master-data=2 -B ${db} > ${DIR}/${db}_`date +%F_%T`.sql
done

find /data/backup -mtime +10 -exec rm -rf {} \;
```

#### 2.5 mysql集群

##### 2.5.1 mysql主从

```bash
############### 概述
# 主从复制相关线程
. 主节点
dump Thread:为每个Slave的I/O Thread启动一个dump线程，用于向其发送binary log events
. 从节点
I/O Thread：向Master请求二进制日志事件，并保存于中继日志中
SQL Thread：从中继日志中读取日志事件，在本地完成重放
# 查看mysql线程
mysql> show processlist;
# 跟复制功能相关的文件
master.info：用于保存slave连接至master时的相关信息，例如账号、密码、服务器地址等
relay-log.info：保存在当前slave节点上已经复制的当前二进制日志和本地relay log日志的对应关系
mysql-relay-bin.00000# : 中继日志,保存从主节点复制过来的二进制日志,本质就是二进制日志
# 主从节点配置要求
主/从尽量使用同一版本
不能满足上述要求，主节点用低版本，从节点用高版本；以满足兼容性要求
```

##### 2.5.2 实现主从配置

```bash
####################主节点配置###################
# 确认主节点是否开启二进制日志,及二进制日志格式
[mysqld]
binlog_format=row
log-bin=/data/mysql_logbin/logbin

mysql> show variables like '%log_bin%';
mysql> show variables like '%binlog_format%';
# 为当前节点设置一个全局惟一的ID号
[mysqld]
server-id=101							#设置唯一server_id,可取服务器IP最后一位，避免重复
#log-basename=master

#查看从二进制日志的文件和位置开始进行复制
mysql> show master status;
mysql> show master logs;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| logbin.000015 |      154 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
#创建有复制权限的用户账号
mysql> create user repluser@'10.0.0.%' identified by '123456';		# 创建账号repluser@'10.0.0.%'；设置密码123456
mysql> grant replication slave on *.* to repluser@'10.0.0.%';		# 对帐号附加权限

####################从节点配置####################
# 启动中继日志
[mysqld]
server-id=104							# 设置唯一server_id,可取服务器IP最后一位，避免重复
binlog_format=row						# 开启二进制日志，并设置格式&存储路径
log-bin=/data/mysql_logbin/logbin
read_only=on							# 设置数据库只读，针对supper user无效
relay_log=relay-log						 # relay log的文件路径，默认值hostname-relay-bin
relay_log_index=relay-log.index
# 使用有复制权限的用户账号连接至主服务器，并启动复制线程
mysql> CHANGE MASTER TO 
    -> MASTER_HOST='10.0.0.101', 
    -> MASTER_USER='repluser', 
    -> MASTER_PASSWORD='123456', 
    -> MASTER_PORT=3306,
    -> MASTER_LOG_FILE='logbin.000001', 
    -> MASTER_LOG_POS=154;
mysql> show slave status\G 				# 查看从节点状态
mysql> start slave;					    # 开启线程
mysql> show processlist;
```

```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.0.0.101
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: logbin.000001
          Read_Master_Log_Pos: 765
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 928
        Relay_Master_Log_File: logbin.000001
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
          Exec_Master_Log_Pos: 765
              Relay_Log_Space: 1129
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
             Master_Server_Id: 101
                  Master_UUID: 6449239b-6a1e-11ed-b038-000c29388cdd
             Master_Info_File: /data/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

#### 2.6 生产环境my.cnf配置案例

```bash
#打开独立表空间
innodb_file_per_table = 1
#MySQL 服务所允许的同时会话数的上限，经常出现Too Many Connections的错误提示，则需要增大此值
max_connections = 8000
#所有线程所打开表的数量
open_files_limit = 10240
#back_log 是操作系统在监听队列中所能保持的连接数
back_log = 300
#每个客户端连接最大的错误允许数量，当超过该次数，MYSQL服务器将禁止此主机的连接请求，直到MYSQL服务器重启或通过flush hosts命令清空此主机的相关信息
max_connect_errors = 1000
#每个连接传输数据大小.最大1G，须是1024的倍数，一般设为最大的BLOB的值
max_allowed_packet = 32M
#指定一个请求的最大连接时间
wait_timeout = 10
# 排序缓冲被用来处理类似ORDER BY以及GROUP BY队列所引起的排序
sort_buffer_size = 16M 
#不带索引的全表扫描.使用的buffer的最小值
join_buffer_size = 16M 
#查询缓冲大小
query_cache_size = 128M
#指定单个查询能够使用的缓冲区大小，缺省为1M
query_cache_limit = 4M    
# 设定默认的事务隔离级别
transaction_isolation = REPEATABLE-READ
# 线程使用的堆大小. 此值限制内存中能处理的存储过程的递归深度和SQL语句复杂性，此容量的内存在每次连接时被预留.
thread_stack = 512K 
# 二进制日志功能
log-bin=/data/mysqlbinlogs/
#二进制日志格式
binlog_format=row
#InnoDB使用一个缓冲池来保存索引和原始数据, 可设置这个变量到物理内存大小的80%
innodb_buffer_pool_size = 24G 
#用来同步IO操作的IO线程的数量
innodb_file_io_threads = 4
#在InnoDb核心内的允许线程数量，建议的设置是CPU数量加上磁盘数量的两倍
innodb_thread_concurrency = 16
# 用来缓冲日志数据的缓冲区的大小
innodb_log_buffer_size = 16M
# 在日志组中每个日志文件的大小
innodb_log_file_size = 512M 
# 在日志组中的文件总数
innodb_log_files_in_group = 3
# SQL语句在被回滚前,InnoDB事务等待InnoDB行锁的时间
innodb_lock_wait_timeout = 120
#慢查询时长
long_query_time = 2
#将没有使用索引的查询也记录下来
log-queries-not-using-indexes
```

