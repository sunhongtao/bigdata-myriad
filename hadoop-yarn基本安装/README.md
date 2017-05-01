####完整记录安装 yarn on mesos  ---myraid 框架
由于git上提供的教程很少，而且因为环境迥异，造成了安装困难。
暂时mesos是跑在宿主机上。zookeeper是容器化。

牵涉甚广 mesos集群的搭建  hadoop集群的搭建和myraid的编译。

配置如下。
20.26.4.41csv-dcos01mesos-master\zookeeper
20.26.4.42csv-dcos02mesos-master\zookeeper\resoucemanager(hadoop)20.26.4.43csv-dcos03mesos-master\zookeeper20.26.4.44csv-dcos04mesos-slave\datanode(hadoop)20.26.4.45csv-dcos05mesos-slave\datanode(hadoop)

版本介绍：
hadoop 2.7.0
mesos  1.1.0
myraid 0.1.0 (由于0.2.0不太成熟，这里选择了0.1.0)
jdk 1.7
前提配置：在各个节点执行。

1、清除sudo的requiretty
使用visudo，注释。

    # Defaults requiretty

2、主机名解析

    [root@csv-dcos02 usr]# cat /etc/hosts
    127.0.0.1   localhost
    ::1 localhost
    
    20.26.4.41csv-dcos01
    20.26.4.42csv-dcos02
    20.26.4.43csv-dcos03
    20.26.4.44csv-dcos04
    20.26.4.45csv-dcos05

###myraid编译

首先去github上下载。
`incubator-myriad-myriad-0.1.0-incubating.tar.gz`

为了加快编译速度。需要配置。

    cat /root/.gradle/gradle.properties   
    org.gradle.daemon=true  // 开启线程守护，第一次编译时开线程，之后就不会再开了
    org.gradle.jvmargs=-Xmx2048m -XX:MaxPermSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8  // 配置编译时的虚拟机大小
    org.gradle.parallel=true  // 开启并行编译，相当于多条线程再走
    org.gradle.configureondemand=true   启用新的孵化模式
    
    [root@localhost incubator-myriad-myriad-0.1.0-incubating]# pwd
    /root/myraid_v1.0/incubator-myriad-myriad-0.1.0-incubating
    [root@localhost incubator-myriad-myriad-0.1.0-incubating]# ./gradlew build

这个真的会编译很久。要1个多小时。在家。而不是在移动。在移动是编译不了的。封的太厉害。
编译好之后。那好多jar文件。就待用了。

接下来就是集成搭建了。
hadoop集群搭建。
这里使用的版本是 2.7.0，需要依赖jdk环境。
1、安装jdk. (csv-dcos02，csv-dcos04，csv-dcos05)
这里使用jdk-7u79-linux-x64.tar.gz。安装在/usr/local/下。
直接解压即可。然后添加环境变量。
在/etc/profile下添加。

    #java
    export JAVA_HOME=/usr/local/jdk1.7.0
    export PATH=$JAVA_HOME/bin:$PATH 
    #hadoop  
    export HADOOP_HOME=/usr/hadoop-2.7.0
    export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
    export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
    export PATH=$HADOOP_HOME/bin:$PATH
    export PATH=$HADOOP_HOME/sbin:$PATH
    export MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
    export PATH=$PATH:/opt/mesosphere/zookeeper/bin
    
    [root@csv-dcos03 ~]# java -version
    java version "1.7.0_79"
    Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
    Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)

成功安装jdk.

###安装hadoop和myraid。
resourcemanager 和nodemanager所需要的myraid的jar是不一样。

一个是exectutor和scheduler。各拷贝所需要的文件到指定hadoop的yarn/lib目录。

下载hadoop-2.7.0.tar.gz。解压至/usr/hadoop-2.7.0下。
[root@csv-dcos02 usr]# ls hadoop-2.7.0/
bin  dfs  etc  include  lib  libexec  LICENSE.txt  logs  NOTICE.txt  README.txt  sbin  share  tmp  tmp-output
[root@csv-dcos02 usr]# pwd
/usr
首先namenode和datanode的安装是不一样的。

首先直接来配置文件 resourcemanager。

    hdfs-site.xml   core-site.xml  hadoop-env.sh yarn-env.sh mapred-site.xml  myraid-config-default.xml

上传jar包和yarn配置到m2:

    [root@localhost incubator-myriad-myriad-0.1.0-incubating]# ls
    build.gradle  DISCLAIMER  docsgradle.properies  gradlew.bat  myriad-commons   myriad-scheduler  README.mdvagrant  website
    configdocker  gradle  gradlew   LICENSE  myriad-executor  NOTICEsettings.gradle  Vagrantfile
    [root@localhost incubator-myriad-myriad-0.1.0-incubating]# pwd
    /root/myraid_v1.0/incubator-myriad-myriad-0.1.0-incubating
    
    cd /root/myraid_v1.0/incubator-myriad-myriad-0.1.0-incubating
    tar -czf libs.tgz myriad-scheduler/build/libs
    scp libs.tgz csv-dcos02:~
    ssh m2 << EOT
    source ~/.bash_profile
      tar -xf libs.tgz
      /bin/cp -f myriad-scheduler/build/libs/*.jar /usr/hadoop-2.7.0/share/hadoop/yarn/lib/
      rm -rf libs.tgz
    EOT
    rm -rf libs.tgz
    
    scp myriad-scheduler/src/main/resources/yarn-site-default.xml csv-dcos02:/usr/local/hadoop/etc/hadoop/yarn-site.xml

具体的配置信息，如下。

    [root@csv-dcos02 hadoop]# cat hdfs-site.xml 
    <?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <!--
      Licensed under the Apache License, Version 2.0 (the "License");
      you may not use this file except in compliance with the License.
      You may obtain a copy of the License at
    
    http://www.apache.org/licenses/LICENSE-2.0
    
      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
      See the License for the specific language governing permissions and
      limitations under the License. See accompanying LICENSE file.
    -->
    
    <!-- Put site-specific property overrides in this file. -->
    <configuration>  
       <property>  
    <name>dfs.namenode.secondary.http-address</name>  
       <value>csv-dcos02:9001</value>  
       </property>  
     <property>  
     <name>dfs.namenode.name.dir</name>  
     <value>file:/usr/hadoop-2.7.0/dfs/name</value>  
       </property>  
      <property>  
      <name>dfs.datanode.data.dir</name>  
      <value>file:/usr/hadoop-2.7.0/dfs/data</value>  
       </property>  
       <property>  
       <name>dfs.replication</name>  
       <value>1</value>  
    </property>  
    <property>  
     <name>dfs.webhdfs.enabled</name>  
      <value>true</value>  
     </property>  
    </configuration>  
    
    
    core-site.xml
    <property>  
    <name>fs.defaultFS</name>  
    <value>hdfs://csv-dcos02:9000</value>  
    </property>  
    <property>  
    <name>io.file.buffer.size</name>  
    <value>131072</value>  
    </property>  
    <property>  
    <name>hadoop.tmp.dir</name>  
    <value>file:/usr/hadoop-2.7.0/tmp</value>  
    <description>Abase for other temporary   directories.</description>  
    </property>
    
    yarn-env.sh 修改
    JAVA=/usr/local/jdk1.7.0/bin/java
    
    mapred-site.xml 
    <configuration>
      <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
      </property>
      <!--This option enables dynamic port assignment by mesos -->
      <property>
    <name>mapreduce.jobhistory.address</name>
    <value>csv-dcos02:10020</value>
    <description>Host and port for Job History Server (default 0.0.0.0:10020)</description>
      </property>
      <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>512</value>
      </property>
    	<property>
    <name>mapreduce.shuffle.port</name>
    <value>${myriad.mapreduce.shuffle.port}</value>
    </property>
    
    cat yarn-site.xml
    <property>
    	<name>yarn.resourcemanager.hostname</name>
    	<value>csv-dcos02</value>
      	</property>
      <!-- Site-specific YARN configuration properties -->
      	<property>
    	<name>yarn.nodemanager.aux-services</name>
    	<value>mapreduce_shuffle,myriad_executor</value>
    <!-- If using MapR distro, please use the following value:
    <value>mapreduce_shuffle,mapr_direct_shuffle,myriad_executor</value> -->
      	</property>
      	<property>
    		<name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
    		<value>org.apache.hadoop.mapred.ShuffleHandler</value>
      	</property>
      	<property>
    		<name>yarn.nodemanager.aux-services.myriad_executor.class</name>
    		<value>org.apache.myriad.executor.MyriadExecutorAuxService</value>
      	</property>
      	<property>
    		<name>yarn.nm.liveness-monitor.expiry-interval-ms</name>
    		<value>2000</value>
      	</property>
      	<property>
    		<name>yarn.am.liveness-monitor.expiry-interval-ms</name>
    		<value>10000</value>
      	</property>
      	<property>
    		<name>yarn.resourcemanager.nm.liveness-monitor.interval-ms</name>
    		<value>1000</value>
      	</property>
      <!-- Configure Myriad Scheduler here -->
      	<property>
    		<name>yarn.resourcemanager.scheduler.class</name>
    		<value>org.apache.myriad.scheduler.yarn.MyriadFairScheduler</value>
    		<description>One can configure other schedulers as well from following list:
    org.apache.myriad.scheduler.yarn.MyriadCapacityScheduler,
    org.apache.myriad.scheduler.yarn.MyriadFifoScheduler
    	</description>
      	</property>
      	<property>
    	<name>yarn.scheduler.minimum-allocation-mb</name>
    	<value>1024</value>
      	</property>
      	<property>
    	<name>yarn.scheduler.minimum-allocation-vcores</name>
    	<value>1</value>
      	</property>
      	<property>
    	<name>yarn.nodemanager.vmem-check-enabled</name>
    	<value>false</value>
      	</property>
    	<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>${nodemanager.resource.cpu-vcores}</value>
    </property>
    <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>${nodemanager.resource.memory-mb}</value>
    </property>
    <!--These options enable dynamic port assignment by mesos -->
    <property>
    <name>yarn.nodemanager.address</name>
    <value>${myriad.yarn.nodemanager.address}</value>
    </property>
    <property>
    <name>yarn.nodemanager.webapp.address</name>
    <value>${myriad.yarn.nodemanager.webapp.address}</value>
    </property>
    <property>
    <name>yarn.nodemanager.webapp.https.address</name>
    <value>${myriad.yarn.nodemanager.webapp.address}</value>
    </property>
    <property>
    <name>yarn.nodemanager.localizer.address</name>
    <value>${myriad.yarn.nodemanager.localizer.address}</value>
    </property>
    <!-- Configure Myriad Scheduler here -->
    <property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.myriad.scheduler.yarn.MyriadFairScheduler</value>
    <description>One can configure other scehdulers as well from following list: org.apache.myriad.scheduler.yarn.MyriadCapacityScheduler, org.apache.myriad.scheduler.yarn.MyriadFifoScheduler</description>
    </property>


[    root@csv-dcos02 hadoop]# grep -v "^#" myriad-config-default.yml 

    mesosMaster: zk://20.26.4.41:2181,20.26.4.42:2181,20.26.4.43:2181/mesos
    checkpoint: false
    frameworkFailoverTimeout: 43200000
    frameworkName: MyriadAlpha
    frameworkRole: 
    frameworkUser: root # User the Node Manager runs as, required if nodeManagerURI set, otherwise defaults to the user
      # running the resource manager.
    frameworkSuperUser: root  # To be depricated, currently permissions need set by a superuser due to Mesos-1790.  Must be
      # root or have passwordless sudo. Required if nodeManagerURI set, ignored otherwise.
    nativeLibrary: /usr/local/lib/libmesos.so
    zkServers: 20.26.4.41:2181,20.26.4.42:2181,20.26.4.43:2181
    zkTimeout: 20000
    restApiPort: 8192
    profiles:
      small:
    cpu: 1 
    mem: 1024
      medium:
    cpu: 2
    mem: 2048
      large:
    cpu: 4
    mem: 4096
    nmInstances: # NMs to start with. Requires at least 1 NM with a non-zero profile.
      medium: 1 # <profile_name : instances>
    rebalancer: false
    haEnabled: false
    nodemanager:
      jvmMaxMemoryMB: 1024
      cpus: 0.2
      cgroups: false
    executor:
      jvmMaxMemoryMB: 256
      #path: file:///usr/local/libexec/mesos/myriad-executor-runnable-0.1.0.jar
      #The following should be used for a remotely distributed URI, hdfs assumed but other URI types valid.
      nodeManagerUri: hdfs://csv-dcos02:9000/dist/hadoop-myriad.tgz
      path: hdfs://csv-dcos02:9000/dist/myriad-executor-runnable-0.1.0.jar
    yarnEnvironment:
      #YARN_HOME: /usr/hadoop-2.7.0
      YARN_HOME: hadoop-2.7.0 #this should be relative if nodeManagerUri is set
      JAVA_HOME: /usr/local/jdk1.7.0 #System dependent, but sometimes necessary
    mesosAuthenticationPrincipal:
    mesosAuthenticationSecretFilename:


添加以下内容到/usr/local/hadoop/etc/hadoop/hadoop-env.sh：

    [root@csv-dcos02 hadoop]# cat hadoop-env.sh  |grep JAVA_HOME
    # The only required environment variable is JAVA_HOME.  All others are
    # set JAVA_HOME in this file, so that it is correctly defined on
    export JAVA_HOME=/usr/local/jdk1.7.0
    export MESOS_NATIVE_JAVA_LIBRARY=/usr/local/lib/libmesos.so


加入masters
echo "csv-dcos02" > /usr/local/hadoop/etc/hadoop/masters
配置slaves

cat > /usr/local/hadoop/etc/hadoop/slaves << EOT
csv-dcos04
csv-dcos05
EOT


在slave节点配置。
cd cd /root/myraid_v1.0/incubator-myriad-myriad-0.1.0-incubating
scp myriad-executor/build/libs/myriad-executor-runnable-0.1.0.jar csv-dcos02:~

（分别拷贝至csv-dcos04,csv-dcos05）
scp  myriad-scheduler/build/libs/*.jar root@csv-dcos04:/usr/hadoop-2.7.0/share/hadoop/yarn/lib/
scp myriad-executor/build/libs/myriad-executor-0.1.0.jar root@csv-dcos04:/usr/hadoop-2.7.0/share/hadoop/yarn/lib/
scp  myriad-scheduler/src/main/resources/myriad-config-default.yml root@csv-dcos04:/usr/hadoop-2.7.0/etc/hadoop/

然后切换到csv-dcos04
cd /usr/
tar -czf hadoop-myriad.tgz hadoop-2.7.0
scp hadoop-myriad.tgz csv-dcos02:~
# put all of this to hdfs
ssh cvs-dcos02 << EOT
  source ~/.bash_profile
  hdfs dfs -mkdir /dist
  hdfs dfs -put -f hadoop-myriad.tgz /dist
  hdfs dfs -put -f myriad-executor-runnable-0.1.0.jar /dist
  rm -rf hadoop-myriad.tgz
  rm -rf myriad-executor-runnable-0.1.0.jar
EOT
同时还要加入变量
*******echo "/usr/local/hadoop" > /etc/mesos-slave/hadoop_home****
否则会提示各种错误。


其中hadoop-env.sh  yarn-env.sh 均需要配置java和mesos的包。 
======================
[root@csv-dcos04 hadoop]# cat mapred-site.xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <!--This option enables dynamic port assignment by mesos -->
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>csv-dcos02:10020</value>
    <description>Host and port for Job History Server (default 0.0.0.0:10020)</description>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>512</value>
  </property>
	<property>
    <name>mapreduce.shuffle.port</name>
    <value>${myriad.mapreduce.shuffle.port}</value>
</property>
	<property>
  <name>mapred.mesos.executor.uri</name>
  <value>hdfs://csv-dcos02:9000/dist/myriad-executor-runnable-0.1.0.jar</value>
</property>
</configuration>

core-site.xml
<property>  
        <name>fs.defaultFS</name>  
        <value>hdfs://csv-dcos02:9000</value>  
    </property>  
    <property>  
        <name>io.file.buffer.size</name>  
        <value>131072</value>  
    </property>  
    <property>  
        <name>hadoop.tmp.dir</name>  
        <value>file:/usr/hadoop-2.7.0/tmp</value>  
        <description>Abase for other temporary   directories.</description>  
    </property>

hdfs-site.xml
  <property>  
              <name>dfs.datanode.data.dir</name>  
              <value>file:/usr/hadoop-2.7.0/dfs/data</value>  
       </property>  
       <property>  
               <name>dfs.replication</name>  
               <value>1</value>  
        </property>  
        <property>  
                 <name>dfs.webhdfs.enabled</name>  
                  <value>true</value>  
         </property>  

以上配置完毕。就可以启动start-dfs.sh。  stop-dfs.sh
mr-jobhistory-daemon.sh start historyserver    mr-jobhistory-daemon.sh stop historyserver   
yarn-daemon.sh start resourcemanager 	yarn-daemon.sh stop resourcemanager

当mesos容器化的时候，会存在mesos所依赖的模块不完全。
当初的做法。是从容器里所有关联的模块都拷贝出来。这个在myraid-config-default.xml里指定了路径。



我做的一些更改，网络上没有陈述的。

[root@csv-dcos02 hadoop]# vi myriad-config-default.yml 

mesosMaster: zk://20.26.4.41:2181,20.26.4.42:2181,20.26.4.43:2181/mesos
checkpoint: false
frameworkFailoverTimeout: 43200000
frameworkName: MyriadAlpha
frameworkRole:
frameworkUser: root     # User the Node Manager runs as, required if nodeManagerURI set, otherwise defaults to the user
                          # running the resource manager.
frameworkSuperUser: root  # To be depricated, currently permissions need set by a superuser due to Mesos-1790.  Must be
                          # root or have passwordless sudo. Required if nodeManagerURI set, ignored otherwise.
nativeLibrary: /usr/local/lib/libmesos.so
zkServers: 20.26.4.41:2181,20.26.4.42:2181,20.26.4.43:2181
zkTimeout: 20000
restApiPort: 8192
profiles:
#  zero:  # NMs launched with this profile dynamically obtain cpu/mem from Mesos
#    cpu: 0
#    mem: 0

测试 wordcount   计数程序。
[root@csv-dcos02 ~]# cat 1.txt 
hello world
haha
my world
my name is sunhongtao
haha  how are you world

[root@csv-dcos02 ~]# hdfs dfs -put 1.txt /test
[root@csv-dcos02 ~]# hdfs dfs -ls /test/
Found 2 items
-rw-r--r--   1 root supergroup         72 2017-03-16 17:16 /test/1.txt

运行
[root@csv-dcos02 ~]#  hadoop jar /usr/hadoop-2.7.0/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar wordcount /test/1.txt /test/output
省略输出mapreduce............................
...................................................................
[root@csv-dcos02 ~]# hdfs dfs -cat /test/output/part-r-00000
are	1
haha	2
hello	1
how	1
is	1
my	2
name	1
sunhongtao	1
world	3
you	1
[root@csv-dcos02 ~]# 

[root@csv-dcos05 hsperfdata_root]# jps
6018 Jps
3715 DataNode
4218 NodeManager







具体的内部调用还需要测试。






