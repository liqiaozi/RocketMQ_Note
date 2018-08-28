---
title:  RocketMQ系列03--双主模式集群搭建(2m-noslave)
date:  2018-08-27 
categories:  RocketMQ 
tags: rocketmq   集群  2m-noslave 
---


# 0.前言 #
　　在上一篇中,我们介绍了 rocketmq 的支持的集群方式,并且介绍了各自的优缺点。本次以多Master集群模式为例搭建一个双机Master的RocketMQ集群环境（**2m-noslave**集群模式）。



# 1.环境准备 #

　　因为rocketmq底层是 Java 语言编写，所以搭建此环境的服务器，必须已经安装和配置了 JDK。

## 　  1.1 工具下载 ##
　　这里，我提供了本次安装rocketmq所用到的工具包。分别是：
![](http://upload-images.jianshu.io/upload_images/11560519-fc516f3822f919d7)



提供了[百度云盘](https://pan.baidu.com/s/1N4QrAgNrdwi3PacqqwxYUg#list/path=%2F%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%B7%A5%E5%85%B7%E5%8C%85%2Frocketmq-software&parentPath=%2F%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA%E5%B7%A5%E5%85%B7%E5%8C%85 "rocketmq安装包")（密码：wdvm）。大家可自行下载。


## 　  1.2 服务器规划  ##
 
| 序号	|        IP      | 角色      					|    模式   |
| :----:|   :---------:  | :------:  					| :------: |
|    1  | 192.168.81.132 | nameServer1,brokerServer1    | Master1  |
|    2  | 192.168.81.134 | nameServer2,brokerServer2    | Master2  |





## 　  1.3 基础环境搭建 ##

1. 将压缩包上传到2台机器

> 我这里使用root用户登录,把软件包放在根目录下。



	
2. 检查是否安装了JDK,如果没有安装过,则先进行安装和配置JDK(参考 [linux下JDK的安装](https://blog.csdn.net/o135248/article/details/79931693))
3.

```
#检查JDK是否已安装
[root@localhost ~]# java -version
java version "1.8.0_11"
Java(TM) SE Runtime Environment (build 1.8.0_11-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.11-b03, mixed mode)
```

# 2.集群搭建　#

> 说明：安装时，请按照自己的实际服务器环境替换相应的IP即可。

## 　  2.1 Hosts 添加信息 ##
这里为了方便我们将 nameserver 和 broker 配置到了一台机器上,可以分开进行配置.



```
#编辑2台机器的hosts文件
vim  /etc/hosts

#配置如下的信息:
192.168.81.132 rocketmq-nameserv1
192.168.81.132 rocketmq-master1

192.168.81.134 rocketmq-nameserv2
192.168.81.134 rocketmq-master2

#重启网卡
service network restart
```

## 　  2.2 解压压缩包(2台机器) ##

```
#解压 alibaba-rocketmq-3.2.6.tar.gz 至 /usr/local
tar -zxvf alibaba-rocketmq-3.2.6.tar.gz -C /usr/local/

#重命名(带上版本号)
mv alibaba-rocketmq alibaba-rocketmq-3.2.6

#创建软连接
ln -s alibaba-rocketmq-3.2.6 rocketmq


#ls看下
rocketmq -> alibaba-rocketmq-3.2.6

```


## 　  2.3 创建存储路径(2台机器) ##

```
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index

#或者执行下面的一条命令：
mkdir -p /usr/local/rocketmq/store/{commitlog,consumequeue,index}
```

## 　  2.4 RocketMQ配置文件(2台机器) ##

```
vim /usr/local/rocketmq/conf/2m-noslave/broker-a.properties
vim /usr/local/rocketmq/conf/2m-noslave/broker-b.properties
```
删除里面的配置，粘贴下面的配置信息。(注意更改brokerName 和 namesrvAddr 的配置)

```
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a|broker-b
#0 表示 Master， >0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=rocketmq-nameserv1:9876;rocketmq-nameserv2:9876

#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911

#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88

#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort

#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000

#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER

#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH

#checkTransactionMessageEnable=false

#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128

```


## 　  2.5 修改日志配置文件(2台机器) ##

```
#创建日志目录logs
mkdir -p /usr/local/rocketmq/logs
#修改日志配置文件
cd /usr/local/rocketmq/conf && sed -i 's#${user.home}#/usr/local/rocketmq#g' *.xml
									
```



## 　  2.6 修改启动脚本参数(2台机器) ##

切换到 /usr/local/devTool/rocketmq/bin 目录下，我们发现这里存在 rocketmq的各种命令（win linux）。启动rocketmq，应该先启动nameserver,再启动broker。而broker的启动需要合适的JVM内存配置，阿里官方推荐的是4G。可以根据实际生产环境进行合理配置。

**编辑文件 runbroker.sh**
```
vim /usr/local/rocketmq/bin/runbroker.sh

#修改 runbroker 的配置 
#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m -XX:PermSize=128m -XX:MaxPermSize=320m"
```

**编辑文件 runserver.sh**

```
vim /usr/local/rocketmq/bin/runserver.sh

#修改配置
#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m -XX:PermSize=128m -XX:MaxPermSize=320m"

```


## 　  2.7 启动NameServer(2台机器) ##

```
cd /usr/local/rocketmq/bin
#后台启动nameserver
nohup sh mqnamesrv &

#查看namesrv日志
tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/namesrv.log

```

## 　  2.8 启动BrokerServer A  ##

```
nohup sh mqbroker -c /usr/local/rocketmq/conf/2m-noslave/broker-a.properties >/dev/null 2>&1 &

#查看进程及日志
jps
tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/broker.log
tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/namesrv.log

```



## 　  2.9 启动BrokerServer B  ##

```
nohup sh mqbroker -c /usr/local/rocketmq/conf/2m-noslave/broker-b.properties >/dev/null 2>&1 &

tail -f -n 500 /usr/local/rocketmq/logs/rocketmqlogs/broker.log
```



## 　  2.10 安装 RocketMq Console ##
1. **解压缩 tomcat 到 /usr/local目录下，并且重命名为 rocketmq-console-tomcat.**
```
#解压缩 tomcat 到 /usr/local目录下
tar -zxvf apache-tomcat-8.5.30.tar.gz -C /usr/local/

#重命名为 rocketmq-console-tomcat
cd /usr/local
mv apache-tomcat-8.5.30/ rocketmq-console-tomcat

```

2. **安装 rocketmq-console.war，修改配置并启动**

我这里将 rocketmq-console安装在【134】的机器上。

```
#将war包放到 刚解压缩的tomcat 的webapps目录下
cp rocketmq-console.war /usr/local/rocketmq-console-tomcat/webapps/

#解压缩war包
unzip rocketmq-console.war -d rocketmq-console

#修改配置
vim /usr/local/rocketmq-console-tomcat/webapps/rocketmq-console/WEB-INF/classes/config.properties 
修改 rocketmq.namesrv.addr 项的内容为：
rocketmq.namesrv.addr = rocketmq-nameserv1:9876;rocketmq-nameserv2:9876

#切换到tomcat的bin路径下，启动tomcat,浏览器访问
cd /usr/local/rocketmq-console-tomcat/bin/
./startup.sh

#启动日志
tail -f -n 500 /usr/local/rocketmq-console-tomcat/logs/catalina.out 

#浏览器访问：（确认防火墙已关闭）
http://192.168.81.134:8080/rocketmq-console
```

效果图：

![](https://upload-images.jianshu.io/upload_images/11560519-f6910b13c30e3f90.png)


## 　  2.11 数据清理 ##

启动时，先启动 nameserver,再启动 broker;

关闭时，则相反，先关闭 broker,再 关闭 nameserver(早起晚归)。
```
cd /usr/local/rocketmq/bin
sh mqshutdown broker
sh mqshutdown namesrv

##等待停止
rm -rf /usr/local/devTool/rocketmq/store
mkdir /usr/local/devTool/rocketmq/store
mkdir /usr/local/devTool/rocketmq/store/commitlog
mkdir /usr/local/devTool/rocketmq/store/consumequeue
mkdir /usr/local/devTool/rocketmq/store/index
```

到此,2m-noslave集群模式已搭建完毕,下一篇,我们会对 broker的配置文件中的配置项做介绍,这样有利于后期的更自如得学习.









