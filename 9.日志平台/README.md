# 系统部署架构
![image](https://github.com/jinyuchen724/linux-base/blob/master/9.日志平台/1.png)

# 系统功能概述

- filebeat用于采集各个应用节点上的日志，并将采集到的日志数据传输到kafka中。
- kafka将获取到的日志信息存储起来，并且作为输入(input)传输给logstash。
- logstash将kafka中的数据作为输入，并且把kafka中的数据进行过滤等其他操作，然后把操作后得到     的数据输入（output）到es(elasticsearch)中。
- es(基于lucene搜索引擎）对logstash中的数据进行处理，通过kibana、grafana等可以对es中的数据进行查询、绘图等。  

# 部署环境

| 主机   |   角色   |   操作系统 |   软件版本  |    备注  |
| ------ | ----- | ----- | ------- | ------ |
| jumpserverhost-1(10.1.1.2)  | server-core,coco | Centos 7.2(x86-64) | 1.3.3.2  |  主节点|
| jumpserverhost-2(10.1.1.3)  | 只需ssh | Centos 7.2(x86-64) |   |  客户节点|

# 架构说明及管理文档

```
http://docs.jumpserver.org/zh/docs/admin_instruction.html
```

# 安装

> 官方的安装文档十分详细,这里不赘述了。
- 根据官方文档安装即可

```
/home/www/kafka/bin/kafka-server-start.sh -daemon /home/www/kafka/config/server-1.properties
```

```
/home/www/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

```
/home/www/kafka/bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
```

```
/home/www/kafka/bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic test
```

```
/home/www/kafka/bin/kafka-console-consumer.sh --bootstrap-server 10.33.98.72:9001 --topic test

/home/www/kafka/bin/kafka-console-producer.sh --broker-list 10.33.98.72:9001 --topic test
```

# 用户使用文档

```
http://docs.jumpserver.org/zh/docs/user_guide.html
```

# API调用


```
http://10.1.1.2/docs/

包括不限于:
1.创建删除节点
2.创建删除资产
3.创建删除对应权限

python调用API
见jump.py
```

# FAQ

- sftp传输文件

使用sftp登录跳板机

```
       #登录端口 #账号   #跳板机地址
$ sftp -P2222 chenjinyu@10.1.1.2
chenjinyu@10.1.1.2's password: 
Please enter 6 digits.
[MFA auth]: 608840       #google验证码

# 显示有权限上传下载文件的主机
sftp>ls
jumpserverhost-2

sftp> cd jumpserverhost-2 #这里是你有权限的资产
sftp> cd SA                                #这里是你的账号目录
sftp> ls                                   #这里可以使用get/put上传下载文件
hosts        hsperfdata_www                                                                                       
initos.log   initosPuppet7.sh                                                                                     
```

- 历史回溯

![image](https://github.com/jinyuchen724/linux-base/blob/master/8.jumpserver/jump1.png)

![image](https://github.com/jinyuchen724/linux-base/blob/master/8.jumpserver/jump2.png)

```

```