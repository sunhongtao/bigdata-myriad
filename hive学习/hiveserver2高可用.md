# hiveserver2高可用 #
----------
hive作为hadoop生态系列中的数据库，尤为重要。可以提供类sql的语句来进行操作数据。


## 集群信息 ##
- 20.26.28.242	mesos-slave	datanode	hive-server
- 20.26.28.243	mesos-slave	datanode	hive-server
- 20.26.28.244	zookeeper\mesos\marathon	journalnode	
- 20.26.28.245	zookeeper\mesos\marathon	journalnode	
- 20.26.28.246	zookeeper\mesos\marathon	journalnode	


## 优势 ##
在生产环境中使用Hive，强烈建议使用HiveServer2来提供服务，好处很多：

1. 在应用端不用部署Hadoop和Hive客户端；
2. 相比hive-cli方式，HiveServer2不用直接将HDFS和Metastore暴漏给用户；
3. 有安全认证机制，并且支持自定义权限校验；
4. 有HA机制，解决应用端的并发和负载均衡问题；
5. JDBC方式，可以使用任何语言，方便与应用进行数据交互；
6. 从2.0开始，HiveServer2提供了WEB UI。

那就开始搞。

## 配置 ##
配置很简单。启动zookeeper集群，然后配置hive-site.xml文件。
在单节点的基础上，需要增加以下配置。

    <property>
	    <name>hive.server2.support.dynamic.service.discovery</name>
	    <value>true</value>
    </property>
     
    <property>
	    <name>hive.server2.zookeeper.namespace</name>
	    <value>hiveserver2_zk</value>
    </property>
     
    <property>
	    <name>hive.zookeeper.quorum</name>
	    <value> 20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181</value>
    </property>
     
    <property>
	    <name>hive.zookeeper.client.port</name>
	    <value>2181</value>
    </property>
     
     
    <property>
	    <name>hive.server2.thrift.bind.host</name>
	    <value>0.0.0.0</value>
    </property>
     
    <property>
	    <name>hive.server2.thrift.port</name>
	    <value>10001</value> //两个HiveServer2实例的端口号要一致
    </property>
	<property>
	    <name>hive.metastore.uris</name>
	    <value>thrift://20.26.28.242:9083,thrift://20.26.28.243:9084</value>
	    <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
	</property>

	<property>
    	<name>hive.server2.thrift.bind.host</name>
    	<value>20.26.28.242</value>
    	<description>Bind host on which to run the HiveServer2 Thrift service.</description>
	</property>


**注意:** 
======================================

1. zookeeper的地址一定要写对。被坑，在茫茫的配置文件中....看晕
2. 尽量只在一台机器上先配置完毕，然后使用**pscp**或者**scp**命令拷贝到各个节点。
3. 根据网上的一些说明，并没有说启动hiveserver之前要启动metastore，被坑....
4. hive.server2.thrift.bind.host 这里不要配置成0.0.0.0,是坑。不然通过jdbc链接的时候，会报错。


## 启动 ##
首先在20.26.28.242上启动metastore

    hive --service metastore

然后启动hiveserver2

	hive --service hiveserver2

首先在20.26.28.243上启动metastore

    hive --service metastore

然后启动hiveserver2

	hive --service hiveserver2

在zookeeper的任意机器上查看znode.可以看到信息已经注册。

    [zk: localhost:2181(CONNECTED) 27] ls /hiveserver2
    [serverUri=csv-dcosstorage56:10000;version=1.1.0-cdh5.8.3;sequence=0000000006, serverUri=csv-dcosstorage57:10000;version=1.1.0-cdh5.8.3;sequence=0000000007]

## 测试 ##

    [root@csv-dcosstorage55 ~]# beeline 
    which: no hbase in (/usr/local/jdk1.7.0/bin:/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/hadoop-2.6.0-cdh5.8.3/sbin:/usr/hadoop-2.6.0-cdh5.8.3/bin:/opt/mesosphere/zookeeper/bin:/usr/hive-1.1.0-cdh5.8.3/bin:/root/bin)
    Beeline version 1.1.0-cdh5.8.3 by Apache Hive
    beeline> !connect jdbc:hive2://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2
    scan complete in 3ms
    Connecting to jdbc:hive2://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2
    Enter username for jdbc:hive2://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2: hive
    Enter password for jdbc:hive2://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2: ****
    17/05/06 15:05:05 [main]: INFO jdbc.HiveConnection: Connected to csv-dcosstorage57:10000
    Connected to: Apache Hive (version 1.1.0-cdh5.8.3)
    Driver: Hive JDBC (version 1.1.0-cdh5.8.3)
    Transaction isolation: TRANSACTION_REPEATABLE_READ
    0: jdbc:hive2://20.26.28.244:2181,20.26.28.24> show databases;
    INFO  : Compiling command(queryId=root_20170506150505_04804cc5-afcd-4aec-9f33-654c189f7aa5): show databases
    INFO  : Semantic Analysis Completed
    INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:database_name, type:string, comment:from deserializer)], properties:null)
    INFO  : Completed compiling command(queryId=root_20170506150505_04804cc5-afcd-4aec-9f33-654c189f7aa5); Time taken: 0.007 seconds
    INFO  : Executing command(queryId=root_20170506150505_04804cc5-afcd-4aec-9f33-654c189f7aa5): show databases
    INFO  : Starting task [Stage-0:DDL] in serial mode
    INFO  : Completed executing command(queryId=root_20170506150505_04804cc5-afcd-4aec-9f33-654c189f7aa5); Time taken: 0.035 seconds
    INFO  : OK
    +----------------+--+
    | database_name  |
    +----------------+--+
    | default|
    | sun|
    | testdb |
    +----------------+--+
    3 rows selected (0.171 seconds)
    0: jdbc:hive2://20.26.28.244:2181,20.26.28.24> 