---
title: Linux常用命令集合
date: 2021-03-07 21:07:42
tags:
---





# 文件传输

## scp

```shell
scp [-1246BCpqrv] [-c cipher] [-F ssh_config] [-i identity_file]
[-l limit] [-o ssh_option] [-P port] [-S program]
[[user@]host1:]file1 [...] [[user@]host2:]file2
```

```shell
scp [可选参数] file_source file_target 
```

- -r： 递归复制整个目录。
- -v：详细方式显示输出。scp和ssh(1)会显示出整个过程的调试信息。这些信息用于调试连接，验证和配置问题。
- -P port：注意是大写的P, port是指定数据传输用到的端口号



```
1、*.tar 用 tar –xvf 解压 
2、*.gz 用 gzip -d或者gunzip 解压 
3、*.tar.gz和*.tgz 用 tar –xzf 解压 
4、*.bz2 用 bzip2 -d或者用bunzip2 解压 
5、*.tar.bz2用tar –xjf 解压 
6、*.Z 用 uncompress 解压 
7、*.tar.Z 用tar –xZf 解压 
8、*.rar 用 unrar e解压 
9、*.zip 用 unzip 解压
```

## unzip解压zip文件

```
unzip [-cflptuvz][-agCjLMnoqsVX][-P <密码>][.zip文件][文件][-d <目录>][-x <文件>] 或 unzip [-Z]
```

- -d<目录> 指定文件解压缩后所要存储的目录。

## tar解压.gz、tar、bz2

这五个是独立的命令，压缩解压都要用到其中一个，可以和别的命令连用但只能用其中一个。

```
-c: 建立压缩档案 
-x：解压 
-t：查看内容 
-r：向压缩归档文件末尾追加文件 
-u：更新原压缩包中的文件
```

下面的参数是根据需要在压缩或解压档案时可选的。

```
-z：有gzip属性的 
-j：有bz2属性的 
-Z：有compress属性的 
-v：显示所有过程 
-O：将文件解开到标准输出
```

下面的参数 -f 是必须的:

```
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。 
```



# 文件夹命令

```
mkdir [-p] dirName

mkdir -p /data/qifei/{data,work,plugins,scripts}
```

- -p 确保目录名称存在，不存在的就建一个。