# 安装配置keepalived

- 20.26.4.42  master
- 20.26.4.43  backup
- 20.26.4.5   vip


## 软件下载地址：

```
http://www.keepalived.org/software/keepalived-1.1.20.tar.gz
```

## 编译keepalived
执行编译，假如出现错误，需要安装openssl-devel

```
yum install openssl-devel
```


安装好之后，keepalived继续编译执行。

```
./configure --prefix=/usr/local/keepalived
```

报错。

```
configure: error: Popt libraries is required
```

出现此错误的原因：

未安装popt的开发包，解决方法：

```
yum install popt-devel
```

安装好popt的开发包，重新./configure 即可。
安装好之后。
如果再有其他错误，找Google。


执行下面命令。

将配置文件拷贝到系统对应的目录下：


    mkdir /etc/keepalived
    cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf
    cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/keepalived
    cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
    cp /usr/local/sbin/keepalived /usr/sbin/


    
    [root@csv-dcos02 ~]# cat /etc/rc.local 
    #!/bin/bash
    # THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
    #
    # It is highly advisable to create own systemd services or udev rules
    # to run scripts during boot instead of using this file.
    #
    # In contrast to previous versions due to parallel execution during boot
    # this script will NOT be run after all other services.
    #
    # Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
    # that this script will be executed during boot.
    
    touch /var/lock/subsys/local
    /usr/local/open-falcon/agent/control start
    /etc/init.d/keepalived start

## 配置keepalived

配置好keepalived.conf之后，主备如下：

### master节点：

    
    [root@csv-dcos02 local]# cat /etc/keepalived/keepalived.conf
    global_defs {  
    router_id NodeA  
    }  
    vrrp_script chk_http_port {  
       script "</dev/tcp/20.26.4.42/8192" # connects and exits  
       interval 5  # check every second  
       weight -2   # default prio: -2 if connect fails  
    }  
    vrrp_instance VI_1 {  
    state MASTER#设置为主服务器  
    interface ens160  #监测网络接口  
    virtual_router_id 51  #主、备必须一样  
    priority 91  #(主、备机取不同的优先级，主机值较大，备份机值较小,值越大优先级越高)  
    advert_int 1   #VRRP Multicast广播周期秒数  
    
    authentication {  
    	auth_type PASS  #VRRP认证方式，主备必须一致  
    	auth_pass 1111   #(密码)  
    }
    
    virtual_ipaddress {  
    20.26.4.5/24 ens160 #VRRP HA虚拟地址  
    }
    track_interface {
    	ens160
    } 
    track_script {
    	chk_http_port
    }
    }
    



### backup节点


    [root@csv-dcos03 keepalived]# cat keepalived.conf 
    global_defs {  
    router_id NodeB  
    }  
    vrrp_script chk_http_port {  
       script "</dev/tcp/20.26.4.43/8192" # connects and exits  
       interval 5  # check every second  
       weight -2   # default prio: -2 if connect fails  
    }  
    vrrp_instance VI_1 {  
    state BACKUP#设置为主服务器  
    interface ens160  #监测网络接口  
    virtual_router_id 51  #主、备必须一样  
    priority 90  #(主、备机取不同的优先级，主机值较大，备份机值较小,值越大优先级越高)  
    advert_int 1   #VRRP Multicast广播周期秒数  
    
    authentication {  
    	auth_type PASS  #VRRP认证方式，主备必须一致  
    	auth_pass 1111   #(密码)  
    }
    
    virtual_ipaddress {  
    20.26.4.5/24 ens160 #VRRP HA虚拟地址  
    }
    track_interface {
    	ens160
    } 
    track_script {
    	chk_http_port
    }
    }


## 设置开机启动

设置keepalived服务

---
###开机启动


    chkconfig keepalived on
    service keepalived start
    

---
###启动服务


```
shell> service keepalived stop
```
###停止服务

```
shell> service keepalived restart
```
###重启服务

如此便配置完成。

