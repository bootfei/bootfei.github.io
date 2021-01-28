---
title: docker install mycat
date: 2021-01-26 11:01:27
tags: [vm]
---


# 修改配置文件

1. 创建配置文件目录存放redis.conf，文件从[官网下载](http://download.redis.io/redis-stable/redis.conf) 或者[github下载](https://github.com/redis/redis/blob/unstable/redis.conf)
2. 创建文件夹,新建配置文件贴入从官网下载的配置文件并修改

```bash
mkdir /usr/local/docker/redis -p
vi /usr/local/docker/redis/redis.conf
```

3. 修改启动默认配置(从上至下依次)

```bash
bind 127.0.0.1 #注释掉这部分，这是限制redis只能本地访问

protected-mode no #默认yes，开启保护模式，限制为本地访问

daemonize no#默认no，改为yes意为以守护进程方式启动，可后台运行，除非kill进程，改为yes会使配置文件方式启动redis失败

databases 16 #数据库个数（可选），我修改了这个只是查看是否生效。。

dir  ./ #输入本地redis数据库存放文件夹（可选）

appendonly yes #redis持久化（可选）
```

# docker启动redis

## 1、快速启动

```css
docker run -itd --name redis-test -p 6379:6379 redis
```

参数说明：

-  -p 6379:6379：映射容器服务的 6379 端口到宿主机的 6379 端口。外部可以直接通过宿主机ip:6379 访问到 Redis 的服务。

## 2、配置文件启动

```bash
docker run -p 6379:6379 --name myredis -v /usr/local/docker/redis/redis.conf:/etc/redis/redis.conf -v /usr/local/docker/redis/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
```

参数解释说明：

- -p 6379:6379 端口映射：前表示主机部分，：后表示容器部分。

- --name myredis  指定该容器名称，查看和进行操作都比较方便。

- -v 挂载目录，规则与端口映射相同。
  
   #挂载配置文件，容器启动成功可以通过更改宿主机的配置文件来达到更改容器实际配置文件的目的
   -v /usr/local/docker/redis/conf/redis.conf:/etc/redis/redis.conf
   
   \#挂载持久化文件
   
   -v /usr/local/docker/redis/data:/data
   /usr/local/my-redis/data是宿主机中持久化文件的位置
   /data/是容器中持久化文件的位置(需要和配置文件中dir属性值一样)，
   **重要： 配置文件映射，docker镜像redis 默认无配置文件。**
   
- -d redis 表示后台启动redis

- redis-server /etc/redis/redis.conf  以配置文件启动redis，加载容器内的conf文件，最终找到的是挂载的目录/usr/local/docker/redis/redis.conf
   **重要:  docker 镜像reids 默认 无配置文件启动**
   
- --appendonly yes  开启redis 持久化

# 查看是否运行成功

```ruby
[root@localhost docker]# docker ps -a | grep redis
602659d1b445        redis                              "docker-entrypoint.s…"   54 minutes ago      Up 54 minutes                  0.0.0.0:6379->6379/tcp   modest_ramanujan
65bb6e6cab9f        redis                              "docker-entrypoint.s…"   About an hour ago   Exited (0) About an hour ago                            myredis
```

