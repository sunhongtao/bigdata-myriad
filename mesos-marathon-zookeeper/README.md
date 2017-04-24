## 安装mesos集群
### 集群分配：
- 20.26.28.244	zookeeper\mesos-master\marathon
- 20.26.28.245	zookeeper\mesos-master\marathon
- 20.26.28.246	zookeeper\mesos-master\marathon
- 20.26.4.41	mesos-slave
- 20.26.4.42	mesos-slave
- 20.26.4.43	mesos-slave
- 20.26.4.44	mesos-slave
- 20.26.4.45	mesos-slave
- 20.26.28.241	mesos-slave
- 20.26.28.242	mesos-slave
- 20.26.28.243	mesos-slave


#### 1、说明
其中mesos采用rpm安装方式，marathon和zookeeper采用容器方式。

#### 2、安装mesos（所有节点）

    rpm -ivh mesos-0.28.0-4.x86_64.rpm

这里会报错。提示安装一些依赖。

    yum install -y subversion-devel protobuf-devel

安装之后，配置mesos-master.service。配置文件如下：

    mesos-master --zk=zk://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/mesos --port=5050 --log_dir=/var/log/mesos --hostname=20.26.28.246 --quorum=2 --work_dir=/var/lib/mesos --cluster=Myriad_mesos

配置mesos-slave.service。配置文件如下：

    mesos-slave --master=zk://20.26.28.244:2181,20.26.28.245:2181,20.26.28.246:2181/mesos --log_dir=/var/log/mesos --containerizers=docker,mesos --hadoop_home=/usr/hadoop-2.6.0-cdh5.8.3 --hostname=20.26.4.42 --ip=20.26.4.42 --isolation=cgroups/cpu,cgroups/mem --work_dir=/var/lib/mesos

此步骤需要在所有个节点上执行。在配置mesos-master.service时，**ip和hostname**注意更改。

然后进行启动。

    systemctl daemon-reload
	systemctl start mesos-master
	systemctl start mesos-slave

#### 3、安装zookeeper,采用容器方式。
启动脚本如下：

    docker run -d \
	--log-opt max-file=10 \
	--log-opt max-size=100m \
	--restart=always --net=host \
	-v /etc/localtime:/etc/localtime:ro \
	-v /data/zookeeper:/data:rw \
	-v /data/logs/zookeeper:/datalog:rw \
	-e MYID=1 \
	-e  ZKENV="server.1=20.26.28.244:2888:3888;server.2=20.26.28.245:2888:3888;server.3=20.26.28.246:2888:3888" \
	20.26.4.41:5000/dcos/dnt-zookeeper-3.4.9:v3.1 \
	sh /home/start_zookeeper.sh

#### 4、启动marathon

    docker run -d \	
	--log-opt max-file=10 \
	--log-opt max-size=100m \
	--restart=always \
	--net=host \
	--name=Myriad-marathon \
	-v /etc/localtime:/etc/localtime:ro \
	-v /data/logs/marathon:/data/logs/marathon:rw \
	-e MYENV=zk://20.26.28.246:2181,20.26.28.245:2181/marathon \
	-e ZKENV=zk://20.26.28.246:2181,20.26.28.244:2181,20.26.28.245:2181/mesos \
	-e MNAME=marathon \
	-e MUSER=dcosadmin \
	-e PASSWORD=zjdcos01 \
	-e MPORT=8080 \
	-e MYIP=20.26.28.245 \
	20.26.28.65:5000/dcos/dnt-marathon-1.3.6:v2.2 \
	sh /home/start_marathon.sh

