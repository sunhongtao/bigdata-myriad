#hive搭建
##首先搭建版本说明：
hive-1.1.0-cdh5.8.3.tar.gz
该版本解压出来之后，没有hive-site.xml。则下载了apache版本的hive，拷贝hive-site.xml。

##metastore搭建
metastore的存储方式采用remote方式。可以在多个终端执行hive操作。

在hadoop生态圈中属于数据仓库的角色。他能够管理hadoop中的数据，同时可以查询hadoop中的数据。

  本质上讲，hive是一个SQL解析引擎。Hive可以把SQL查询转换为MapReduce中的job来运行。
  hive有一套映射工具，可以把SQL转换为MapReduce中的job，可以把SQL中的表、字段转换为HDFS中的文件(夹)以及文件中的列。
  这套映射工具称之为metastore，一般存放在derby、mysql中。

###安装mysql
  ● CentOS 7 默认yum源并未包含mysql，需要首先配置repo源：

    $ wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
  ● 安装mysql-community-release-el7-5.noarch.rpm包

    $ sudo rpm -ivh mysql-community-release-el7-5.noarch.rpm
  ● 安装mysql并开启mysql

    $ sudo yum install mysql-server
  ● 为当前用户分配mysql权限并重启mysql服务
    $ sudo chown -R root:root /var/lib/mysql
    $ service mysqld restart

###安装mysql connector

    $ sudo yum install mysql-connector-java
    $ ln -s /usr/share/java/mysql-connector-java.jar /usr/lib/hive/lib/mysql-connector-java.jar

###创建数据库及用户
（1）创建数据库：
    
    [zero@CentOS-StandAlone ~]$ mysql -u root -p
    Enter password:
    ......
    mysql> CREATE DATABASE metastore;
    Query OK, 1 row affected (0.11 sec)
     
    mysql> USE metastore;
    Database changed
     
    ------------------------------------------------------------------------------------------------------
 （2）创建hive用户并授权
    
    mysql> CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
    Query OK, 0 rows affected (0.05 sec)
     
    mysql> CREATE USER 'hive'@'20.26.28.242' IDENTIFIED BY 'hive';
    Query OK, 0 rows affected (0.04 sec)
     
    mysql> CREATE USER 'hive'@'csv-dcosstorage56' IDENTIFIED BY 'hive';
    Query OK, 0 rows affected (0.01 sec)
     
    mysql> REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'hive'@'%';
    Query OK, 0 rows affected (0.00 sec)
     
    mysql> GRANT SELECT,CREATE,INDEX,TEINSERT,UPDATE,DELETE,LOCK TABLES,EXECUTE ON metastore.* TO 'hive'@'%';
    Query OK, 0 rows affected (0.03 sec)
     //必须添加create,index权限，不然会报错。
    mysql> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.02 sec)
    
    ALTER DATABASE metastore CHARACTER SET latin1;
    更改字符集为latin1;
    
    通过yum安装mysql.有个好处可以直接安装mysql-connector.
    $ sudo yum install mysql-connector-java
    $ ln -s /usr/share/java/mysql-connector-java.jar /usr/lib/hive/lib/mysql-connector-java.jar


###配置 Hive

  ● 创建Hive相关文件的存储路径并更改目录权限

    hdfs dfs -mkdir -p /usr/hive/warehouse
    hdfs dfs -mkdir -p /usr/hive/tmp
    hdfs dfs -mkdir -p /usr/hive/log
    hdfs dfs -chmod g+w /usr/hive/warehouse
    hdfs dfs -chmod g+w /usr/hive/tmp
    hdfs dfs -chmod g+w /usr/hive/log

  ● 配置环境变量

    $ sudo vim /usr/lib/hive/conf/hive-env.sh
    export HADOOP_HOME=/usr/lib/hadoop
    export HIVE_CONF_DIR=/usr/lib/hive/conf

格式化数据库

    $ cd /usr/lib/hive/bin$ ./schematool --dbType mysql --initSchema

可能报一些错误。度娘解决之。

###启动metastore服务
必须先启动metastore服务，再启动hive

    [hadoop@bigdata-senior03 hive-0.13.1-cdh5.3.6]$ hive --service metastore
将metastore服务设置为守护进程

    [hadoop@bigdata-senior03 hive-0.13.1-cdh5.3.6]$ nohup hive --service metastore > hive_metastore.run.log 2>&1 &
    

###碰到的错误


1. 解决方法：下载hadoop-mapreduce-client-common-2.6.0-cdh5.4.4.jar 替换hadoop-mapreduce里的包


1. 在配置文件hive-site.xml里找"system:java.io.tmpdir"把他们都换成绝对路径如：/usr/hive-1.1.0-cdh5.8.3/iotmp

1. 执行hive报错。
2. 
    Error during job, obtaining debugging information...
    FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask
    通过yarn界面作业的报错，看是这样的。
    REDUCE capability required is more than the supported max container capability in the cluster. Killing the Job. reduceResourceRequest: <memory:3840, vCores:1> maxContainerCapability:<memory:2048, vCores:2>
    
    增加单个nodemanager的配置。 通过myriad来启动。

###打包hive_UI

下载源代码，根据自己的hive版本下载

`http://archive.cloudera.com/cdh5/cdh/5/hive-1.1.0-cdh5.8.3-src.tar.gz`

2. 解压后将 ./hwi/web/ 目录打包成 hive-hwi-**.war 放入放到hive/lib目录下，打包方式是 执行下面的语句
3. 
    jar cvfM0 hive-hwi-1.1.0.war -C web/ .
    
    <property>
      <name>hive.hwi.war.file</name>
      <value>lib/hive-hwi-1.2.1.war</value>
      <description>This is the WAR file with the jsp content for Hive Web Interface</description>
    </property>
 
运行hive -ui

hive --service hwi


**
执行hiveql语句。看后端调用。


原因，没配置yarn.  在mapreduce-site.xml**



===============================
###启动hive服务

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

    hive --service hiveserver 50031（端口号）
    


启动hive客户端
直接执行hive
或者打开网页客户端。

###HWI与CLI对比（网上摘抄）
如果使用过cli的朋友看了上面的介绍，一定会发现一个很严重的问题：执行的过程没有提示。我们不知道某一个查询执行是什么时候结束的。

总结一下HWI与CLI对比的优缺点：

优点：HWI支持浏览器的方式浏览，方便直观。

缺点：无执行过程提示。

我个人还是更倾向于使用cli的方式:)





