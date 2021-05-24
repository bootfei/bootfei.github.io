---
title: linux安装nginx
date: 2020-11-07 15:28:54
tags: [nginx]
---



1. 基础的操作教程，有yum仓库操作的介绍
   https://www.cyberciti.biz/faq/how-to-install-and-use-nginx-on-centos-7-rhel-7/
2. Nginx official tutorial
   https://www.nginx.com/resources/wiki/start/topics/tutorials/install/



### 安装步骤

#### 命令

```shell
sudo yum install nginx #安装nginx
sudo systemctl start nginx #启动nginx
sudo nginx -t #检查Nginx配置文件是否有语法错误
sudo nginx -s reload #重启
```

#### 修改端口和静态web资源路径

```
vim /etc/nginx/nginx.conf
```



```
server {
		listen 8001 default_server; #从80端口改到8001
		
		root	/home/qf/dist; #资源路径从/user/share/nginx/html改到自定义的文件目录/home/qf/dist
}
```



### 检查步骤

#### 检查启动nginx的user

![img](https://img-blog.csdn.net/20170718094235060?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvb25seXN1bm55Ym95/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 检查是否缺少index文件

就是配置文件中index index.html index.htm这行中的指定的文件。

```
    server {  

      listen       80;  

      server_name  localhost;  

      index  index.php index.html;  

      root  /data/www/;

    }
```

#### 检查权限

修改web目录的读写权限，或者是把nginx的启动用户改成目录的所属用户，重启Nginx即可解决

```
 chmod -R 777 /data/www/
```

