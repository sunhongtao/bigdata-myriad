#hive高可用搭建和使用#
为了验证myriad的使用，必须搭建hadoop整个生态系列。

##版本##
采用版本为：hive-1.1.0-cdh5.8.3.tar.gz
##高可用方案##
高可用采用mesos搭建中采用的zookeeper集群
##下载##
可以CDH的官网下载。

[http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.8.3.tar.gz](http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.8.3.tar.gz "hive-1.1.0-cdh5.8.3.tar.gz下载地址")

##安装##
解压到/usr/目录下即可，需要配置/etc/profile.
    
    export HIVE_CONF_DIR=/usr/hive-1.1.0-cdh5.8.3/conf
    export PATH=$PATH:$HIVE_HOME/bin

##配置##
配置文件hive-env.sh中，需要指定hadoop的目录和java的目录。

        # Set HADOOP_HOME to point to a specific hadoop install directory
    	HADOOP_HOME=/usr/hadoop-2.6.0-cdh5.8.3
    	# Hive Configuration Directory can be controlled by:
     	export HIVE_CONF_DIR=/usr/hive-1.1.0-cdh5.8.3/conf

配置文件hive-site.xml的配置，详情见hive-site.xml。

##高可用配置##

    <property>
    <name>spark.deploy.recoveryMode</name>
    <value>ZOOKEEPER</value>
    </property>
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
    <value>20.26.25.244:2181,20.26.25.245:2181,60.26.25.244:2181</value>
    </property>
    <property>
    <name>hive.zookeeper.client.port</name>
    <value>2181</value>
    </property>

##测试连接JDBC

JDBC连接的URL规范：

    jdbc:hive2://<zookeeper quorum>/<dbName>;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=nameSpace

语法解析:

    <zookeeper quorum> # 为Zookeeper的集群链接串，如zkNode1:2181,zkNode2:2181,zkNode3:2181
    <dbName>   # 为Hive数据库(可不填, 默认为default)
    serviceDiscoveryMode=zooKeeper # 指定模式为zooKeeper
    zooKeeperNamespace=nameSpace   # 指定ZK中的nameSpace，即参数hive.server2.zookeeper.namespace所定义，在hive-site.sh中定义为hiveserver2_zk

##启动

    启动hive服务
    启动hive必须要配置hive-metastore.
    基于metastore可以启动多个client的方法
    然后在服务端启动metastore即可:
    启动metastore:
    bin/Hive --service metastore
    后台启动:
    bin/hive --service metastore 2>&1 >> /var/log.log &
    后台启动,关闭shell连接依然存在:
    nohup bin/hive --service metastore 2>&1 >> /var/log.log &
    
    启动hive server
    hive --service hiveserver2 50031（端口号）


