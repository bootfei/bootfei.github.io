---
title: elastic-安装
date: 2021-03-08 17:05:46
tags:
---



# **Elasticsearch** 单机安装

## 下载安装包

```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.6.2.tar.gz
```

## 解压缩并且修改配置文件

```shell
tar -zxf elasticsearch-6.6.2.tar.gz

cd elasticsearch-6.6.2


```

<font color='red'>注意：把elasticsearch软件必须放入/home/es（es是新建用户）的目录下，并把elasticsearch设置为es用户所属</font>

## 修改Linux配置

### max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

每个进程最大同时打开文件数太小，可通过下面2个命令查看当前数量

```
ulimit -Hn

ulimit -Sn
```

修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效

```
* soft nofile 65536

* hard nofile 65536
```

### max number of threads [3818] for user [qf] is too low, increase to at least [4096]

可通过命令查看

```
ulimit -Hu

ulimit -Su
```

问题同上，最大线程个数太低。修改配置文件/etc/security/limits.conf，增加配置

```
* soft nproc 4096

* hard nproc 4096
```

### max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

修改/etc/sysctl.conf文件

```
vi /etc/sysctl.conf

sysctl -p #执行命令sysctl -p生效
```

增加配置

```
vm.max_map_count=262144
```

错误解决完毕：重新启动

## 修改ES配置

### **创建日志、数据存储目录：（留作备用，初次先创建）**

```
mkdir -p /data/logs/es
mkdir -p /data/es/{data,work,plugins,scripts}
```

### **创建用户**

```
useradd es -s /bin/bash #es不能在root用户下启动，必须创建新的用户，用来启动es
```

注意：es不能在root用户下启动，必须创建新的用户，用来启动es

### **切换用户**

```
su es
```

再次启动，发现还是报错，原因：当前用户没有执行权限

### 授权

```
chown -R es /home/qifei/elasticsearch-6.2.4
```

### 允许外网访问

授权成功，发现elasticsearch已经在es用户下面了，可以启动了，但是启动成功，浏览器不能访问，因此还需要做如下配置：

```
vi confifig/Elasticsearch.yml

network.host: 0.0.0.0 #设置外网访问，默认禁止外网访问
```



### 修改ES的JVM大小

vi config/jvm.options

-Xms1g

-Xmx1g

必须保持一样大

> \* soft nofile 65536
>
> \* hard nofile 65536
>
> \* soft nproc 4096
>
> \* hard nproc 4096

### 启动

不要root启动

```
useradd qf
echo "Password" | passwd qf --stdin
```



```
bin/elasticsearch

bin/elasticsearch -d #后台启动（守护模式）
```



### 验证环境 

not run elasticsearch as root 

curl http://localhost:9200/?pretty

# 分词器

下载后解压到elasticsearch的plugins目录下

POST /_analyze { "analyzer": "simple", "text": " 我是中国人" }

## ik分词器

wget https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v6.6.2

```
POST /_analyze 
{ 
    "analyzer": "ik_smart", 
    "text": "我是中国人 " 
}
```



## pinyin分词器

wget https://github.com/medcl/elasticsearch-analysis-pinyin/releases/tag/v6.6.2

```
GET /medcl/_analyze
{
    "text":["河南濮阳"],
    "analyzer": "pinyin_analyzer"
}
```



# Kibana

## 下载安装

wget https://artifacts.elastic.co/downloads/kibana/kibana-6.6.2-linux-x86_64.tar.gz

## 修改配置文件

```
vi kibana-6.4.2-linux-x86_64/config/kibana.yml

server.port: 5601 ## Kibana服务端口

server.host: "0.0.0.0" ## Kibana服务地址

elasticsearch.url: "http://localhost:9200" ##elasticsearch服务地址
```

## 启动

```shell
./bin/kibana

#后台启动
./bin/kibana & disown 
```

## 验证环境

访问http://192.168.199.216:5601/app/kibana

如果失败，关闭防火墙



# HeadNode安装

# **head**插件主要用途

elasticsearch-head是一个用来浏览、与Elastic Search簇进行交互的web前端展示插件。

elasticsearch-head是一个用来监控Elastic Search状态的客户端插件。

elasticsearch主要有以下三个主要操作——

1）簇浏览，显示簇的拓扑并允许你执行索引（index)和节点层面的操作。

2）查询接口，允许你查询簇并以原始json格式或表格的形式显示检索结果。

3）显示簇状态，有许多快速访问的tabs用来显示簇的状态。 

4）支持RestfulAPI接口，包含了许多选项产生感兴趣的结果，包括： 

第一，请求方式:get,put,post,delete;json请求数据，节点node， 路径path。 

第二，JSON验证器。 

第三，定时请求的能力。

第四，用javascript表达式传输结果的能力。 

第五，统计一段时间的结果或该段时间结果比对的能力。

第六，以简单图标的形式绘制传输结果

## **安装**

安装步骤：

```
#下载nodejs,head插件运行依赖node
wget https://nodejs.org/dist/v9.9.0/node-v9.9.0-linux-x64.tar.xz

#解压
tar -xf node-v9.9.0-linux-x64.tar.xz

#重命名
mv node-v9.9.0-linux-x64 nodeJs

#配置文件
vim /etc/profile

#刷新配置
source /etc/profile

#查询node版本，同时查看是否安装成功
node -v

#下载head插件
wget https://github.com/mobz/elasticsearch-head/archive/master.zip

#解压
unzip master.zip

#使用淘宝的镜像库进行下载，速度很快

npm install -g cnpm --registry=https://registry.npm.taobao.org

#进入head插件解压目录，执行安装命令
cnpm install
```

