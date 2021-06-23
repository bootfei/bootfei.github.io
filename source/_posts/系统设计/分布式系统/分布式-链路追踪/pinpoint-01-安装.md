---
title: pinpoint-01-安装
date: 2021-06-22 19:45:45
tags:
---

### HBase

Download, Configure, and Start HBase

```
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/2.3.5/hbase-2.3.5-bin.tar.gz
$ tar xzvf hbase-x.x.x-bin.tar.gz
$ cd hbase-x.x.x/
$ ./bin/start-hbase.sh
```

*通过$HBASE_HOME/conf/hbase-env.sh文件设置一些环境变量：*

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home
export HBASE_OPTS="-XX:+UseConcMarkSweepGC"
export SERVER_GC_OPTS="-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/Users/zhuzhi/hbase124/logs/jdk8.log"
```

> 如何找到jdk安装路径
>
> ```
> which java
> ls -lrt /usr/bin/java
> ls -lrt /etc/alternatives/java
> ```

See [scripts](https://github.com/pinpoint-apm/pinpoint/tree/master/hbase/scripts) and Run.

```
$ ./bin/hbase shell hbase-create.hbase
```



### Pinpoint Collector

```bash
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.2.2/pinpoint-collector-boot-2.2.2.jar

java -jar -Dpinpoint.zookeeper.address=localhost pinpoint-collector-boot-2.2.2.jar
```





### Pinpoint Web

```
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.2.2/pinpoint-web-boot-2.2.2.jar

java -jar -Dpinpoint.zookeeper.address=localhost pinpoint-web-boot-2.2.2.jar
```



### Java Agent

#### Requirements

In order to build Pinpoint, the following requirements must be met:

- JDK 8 installed

#### 下载

```bash
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.2.2/pinpoint-agent-2.2.2.tar.gz

tar xvzf pinpoint-agent-2.2.2.tar.gz
```



#### 使用官方的quick-start项目测试

Download Pinpoint with `git clone https://github.com/pinpoint-apm/pinpoint.git` or [download](https://github.com/pinpoint-apm/pinpoint/archive/master.zip) the project as a zip file and unzip.

Change to the pinpoint directory, and build.

```
$ cd pinpoint
$ ./mvnw install -DskipTests=true 
```

Change to the quickstart testapp directory, and build. Let’s build and run.

```
$ cd quickstart/testapp
$ ./mvnw clean package
```

#### 运行

Change to the pinpoint directory, and run.

```shell
$ cd ../../
$ java -jar -javaagent:agent/target/pinpoint-agent-2.2.2/pinpoint-bootstrap.jar -Dpinpoint.agentId=test-agent -Dpinpoint.applicationName=TESTAPP quickstart/testapp/target/pinpoint-quickstart-testapp-2.2.2.jar
```

Spring Boot’s embedded Apache Tomcat server is acting as a webserver and is listening for requests on localhost port 8082. Open your browser and in the address bar at the top enter http://localhost:8082

