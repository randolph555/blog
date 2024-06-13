数据同步方向 ：mysql->starRocks 

参考资料：
https://nightlies.apache.org/flink/flink-cdc-docs-release-3.1/zh/docs/get-started/quickstart/mysql-to-starrocks/
https://docs.starrocks.io/docs/loading/Flink_cdc_load/

查看java版本
java -version
下载并解压 [Flink](https://flink.apache.org/downloads.html)
cd /opt
wget https://archive.apache.org/dist/flink/flink-1.18.0/flink-1.18.0-bin-scala_2.12.tgz

如果apache太慢，改为清华源
wget https://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-1.18.1/flink-1.18.1-bin-scala_2.12.tgz
解压文件
tar -xzf flink-1.18.0-bin-scala_2.12.tgz
进入Flink目录
cd flink-1.18.0
通过在 conf/flink-conf.yaml 配置文件追加下列参数开启 checkpoint，每隔 3 秒做一次 checkpoint。
execution.checkpointing.interval: 3000 #生产环境改为5-10分钟
#启动Flink集群
#启动集群
./bin/start-cluster.sh

#停止集群
./bin/stop-cluster.sh

启动成功的话，可以在 http://localhost:8081/ 访问到 Flink Web UI

通过 Flink CDC CLI 提交任务：
1. 下载下面列出的二进制压缩包，并解压得到目录 flink-cdc-3.1.0；
[flink-cdc-3.1.0-bin.tar.gz](https://www.apache.org/dyn/closer.lua/flink/flink-cdc-3.1.0/flink-cdc-3.1.0-bin.tar.gz) flink-cdc-3.1.0 下会包含 bin、lib、log、conf 四个目录。
wget https://www.apache.org/dyn/closer.lua/flink/flink-cdc-3.1.0/flink-cdc-3.1.0-bin.tar.gz
tar -xzf flink-cdc-3.1.0-bin.tar.gz

同样如果太慢，清华源
wget https://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-cdc-3.1.0/flink-cdc-3.1.0-bin.tar.gz
2. 下载下面列出的 connector 包，并且移动到 lib 目录下； 下载链接只对已发布的版本有效, SNAPSHOT 版本需要本地基于 master 或 release- 分支编译。
  - [MySQL pipeline connector 3.1.0](https://search.maven.org/remotecontent?filepath=org/apache/flink/flink-cdc-pipeline-connector-mysql/3.1.0/flink-cdc-pipeline-connector-mysql-3.1.0.jar)
  - [StarRocks pipeline connector 3.1.0](https://search.maven.org/remotecontent?filepath=org/apache/flink/flink-cdc-pipeline-connector-starrocks/3.1.0/flink-cdc-pipeline-connector-starrocks-3.1.0.jar)
wget https://search.maven.org/remotecontent?filepath=org/apache/flink/flink-cdc-pipeline-connector-mysql/3.1.0/flink-cdc-pipeline-connector-mysql-3.1.0.jar
wget https://search.maven.org/remotecontent?filepath=org/apache/flink/flink-cdc-pipeline-connector-starrocks/3.1.0/flink-cdc-pipeline-connector-starrocks-3.1.0.jar

服务器下载太慢的话下载到本地，scp吧
scp ~/Downloads/flink-cdc-pipeline-connector-mysql-3.1.0.jar volcengine-121:/opt/flink-cdc-3.1.0/lib
scp ~/Downloads/flink-cdc-pipeline-connector-starrocks-3.1.0.jar volcengine-121:/opt/flink-cdc-3.1.0/lib

3. 您还需要将下面的 Driver 包放在 Flink lib 目录下，或通过 --jar 参数将其传入 Flink CDC CLI，因为 CDC Connectors 不再包含这些 Drivers：
  - [MySQL Connector Java](https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar)
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.27/mysql-connector-java-8.0.27.jar

下载本地scp,注意使用mysql-connector-java-8.0.28.jar
scp ~/Downloads/mysql-connector-java-8.0.28.jar volcengine-121:/opt/flink-1.18.1/lib
scp ~/Downloads/mysql-connector-java-8.0.28.jar volcengine-121:/opt/flink-cdc-3.1.0/lib
      如果 Flink 已经处于运行状态中，则需要先停止 Flink，然后重启 Flink ，以加载并生效 JAR 包。
./bin/stop-cluster.sh
 ./bin/start-cluster.sh

      配置FLINK_HOME，后面启动CDC的时候需要使用到
export FLINK_HOME=/opt/flink-1.18.1
开启 MySQL Binlog 日志
编辑 MySQL 配置文件 my.cnf（默认路径为 /etc/my.cnf），以开启 MySQL Binlog。
# 开启 Binlog 日志
log_bin = ON
# 设置 Binlog 的存储位置
log_bin =/var/lib/mysql/mysql-bin
# 设置 server_id 
# 在 MySQL 5.7.3 及以后版本，如果没有 server_id，那么设置 binlog 后无法开启 MySQL 服务 
server_id = 1
# 设置 Binlog 模式为 ROW
binlog_format = ROW
# binlog 日志的基本文件名，后面会追加标识来表示每一个 Binlog 文件
log_bin_basename =/var/lib/mysql/mysql-bin
# binlog 文件的索引文件，管理所有 Binlog 文件的目录
log_bin_index =/var/lib/mysql/mysql-bin.index
执行如下命令，重启 MySQL，生效修改后的配置文件：
 # 使用 service 启动
 service mysqld restart
 # 使用 mysqld 脚本启动
 /etc/init.d/mysqld restart
连接 MySQL，执行如下语句确认是否已经开启 Binlog：
-- 连接 MySQL
mysql -h xxx.xx.xxx.xx -P 3306 -u root -pxxxxxx

-- 检查是否已经开启 MySQL Binlog，`ON`就表示已开启
mysql> SHOW VARIABLES LIKE 'log_bin'; 
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)
编写任务配置 yaml 文件。 下面给出了一个整库同步的示例文件 mysql-to-starrocks.yaml：
################################################################################
# Description: Sync MySQL all tables to StarRocks
################################################################################
source:
  type: mysql
  hostname: 127.0.0.1
  port: 3306
  username: root
  password: root
  tables: app_db.\.*
  server-id: 1       #server-id 查看mysql的配置
  server-time-zone: Asia/Shanghai   #注意这个时区要和mysql的时区一致

sink:
   type: starrocks
   name: StarRocks Sink
   jdbc-url: jdbc:mysql://127.0.0.1:9030
   load-url: 127.0.0.1:8030
   username: root
   password: ""
   table.create.properties.replication_num: 1  #StarRocks BE 节点数


route:
  - source-table: app_db.orders
    sink-table: ods_db.ods_orders
  - source-table: app_db.shipments
    sink-table: ods_db.ods_shipments
  - source-table: app_db.products
    sink-table: ods_db.ods_products

pipeline:
  name: Sync MySQL Database to StarRocks
  parallelism: 2
验证语句
select VERSION();
SHOW VARIABLES LIKE 'log_bin'; 
SHOW VARIABLES LIKE 'server_id';

通过命令行提交任务到 Flink Standalone cluster
bash bin/flink-cdc.sh mysql-to-starrocks.yaml

在 Flink Web UI，可以看到一个名为 Sync MySQL Database to StarRocks 的任务正在运行。
http://localhost:8081/

[图片]

测试数据
INSERT INTO app_db.orders (id, price) VALUES (3, 100.00);

ALTER TABLE app_db.orders ADD amount varchar(100) NULL;

UPDATE app_db.orders SET price=100.00, amount=100.00 WHERE id=1;

DELETE FROM app_db.orders WHERE id=2;