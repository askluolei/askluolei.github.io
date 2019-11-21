---
title: centos 安装 ES
tags:
  - 安装
  - ES
categories:
  - 技术笔记
date: 2019-11-22 00:47:23
---


记录一下 `centos7` 下的 `ES` 安装 
首先需要安装 `jdk`  
`centos7` 里面默认安装了 `openjdk` 替换为 `hotspot` 的 `jdk`  
首先卸载  
```sh

sudo rpm --nodeps $(rpm -qa | grep java)
sudo rpm --nodeps $(rpm -qa | grep jdk)
```

然后使用一下命令，没有 `java` 信息，就说明卸载干净了  
```sh
rpm -qa|grep java
rpm -qa|grep jdk
```

下载 `rpm` 的 `jdk` 包，可以去华为的镜像站去下载  
```sh
wget https://repo.huaweicloud.com/java/jdk/8u202-b08/jdk-8u202-linux-x64.rpm
``` 

然后安装  
```sh
sudo rpm -ivh jdk-8u202-linux-x64.rpm
```

验证是否安装成功  
```sh
java -version
```

现在，需要下载 `ES` 了  
```sh
wget https://mirrors.huaweicloud.com/elasticsearch/6.8.5/elasticsearch-6.8.5.tar.gz
```

解压
```sh
tar zxvf elasticsearch-6.8.5.tar.gz
```

到 bin 目录启动 
```sh
cd elasticsearch-6.8.5/bin
./elasticsearch
```

可以看到启动日志，当然，也可能启动失败，`ES` 不能用 `root` 用户启动，因此，需要添加一个普通用户 
```sh
groupadd elsearch
useradd elsearch -g elsearch -p elasticsearch
```

然后需要修改 `elasticsearch` 文件夹的所属用户,注意执行命令的目录，在解压的 `elasticsearch-6.8.5` 父目录
```sh
sudo chown -R elsearch:elsearch elasticsearch
```

然后切换到 `elsearch` 用户再启动  
```sh
su elsearch
./elasticsearch
```

想后台启动，使用  
```sh
./elasticsearch -d
```

验证 `ES` 启动是否成功，访问  
```sh
curl 'http://localhost:9200?pretty'
```

看到返回结果，就是启动成功了  

目前 `ES` 只能本机访问，如果需要外网访问，需要修改配置  `config/elasticsearch.yml`  
```yml
network.host: 0.0.0.0
```
或者使用指定的 `ip`

然后启动。  
可能会报错，因为 ES 这时候开启了启动检测。一些系统参数可能需要修改
```
sudo sysctl -w vm.max_map_count=262144
```



