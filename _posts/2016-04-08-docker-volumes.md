---
layout: post
title: Docker-构建Java应用服务
subtitle: 使用Docker容器构建Tomcat服务器
author: carm
date: 2016-04-08 18:06:21 +0800
header-img: img/home-bg-city.jpg
categories: 技术
tags:
    - Docker
---

# Docker数据卷应用---构建Java应用服务

## 关于Docker
Docker容器是一种轻量级的虚拟技术，可以快速搭建好开发和运行环境，并且该环境可以各种平台上运行。这就为测试和产品部署节省了大量的人力和时间。目前，由官方维护的Docker Hub上已经涌现出大量的基础镜像，读者可以去查看[Docker Hub](https://hub.docker.com/)。不过由于国内GWF的原因，从Docker Hub下载镜像显然没那么容易。

经过一上午亲自实践后，偷着午休时间记录下学习过程，以防遗忘。

## 关于数据卷
数据卷是一个可供一个或多个容器使用的特殊目录，使用它可以达到以下效果

* 数据卷在容器间共享和重用
* 数据卷在宿主机和容器之间共享目录
* 数据卷在宿主和容器间共享单个文件
* 对数据卷的修改不会对镜像产生影响
* 数据卷一直存在直到没有任何容器使用它

> 本文中的所有操作都需使用root权限，ubuntu下可使用sudo来获取root权限

### 从Docker Hub下载tomcat：latest镜像
官方链接：[docker hub](https://hub.docker.com/)

    $ docker pull tomcat

### 挂载宿主机目录作为数据卷
    $ mkdir -p $HOME/tomcat/webapps
    $ docker run -d -p 8080:8080 \
      -v $HOME/tomcat/webapps:/usr/local/tomcat/webapps \
      --name tomcat1 tomcat
    # -d: --detach 后台运行
    # -p: 宿主主机端口：容器端口
    # -v: 宿主主机目录：容器内目录
    # --name: 为容器命名为tomcat1   
	# tomcat: tomcat镜像
    # 通过localhost:8080测试下tomcat是否启动成功
    # 将project.WAR文件放入$HOME/tomcat/webspps下，测试localhost:8080/project

### 多个容器共享数据卷    
    # 创建容器tomcat2,使用容器tomcat1的数据卷
    $ docker run -d -p 8888:8080 --columes-from=tomcat1 --name tomcat2 tomcat

    # 创建新容器tomcat2，引用tomcat1的数据卷。即tomcat1与tomcat2容器使用的是同一数据卷/usr/local/tomcat/webapps
    # 测试：localhost:8888/project
### 查看容器配置 `docker inspect tomcat2`
    # 部分配置如下：
    "VolumesFrom": [
                "tomcat1"
            ]
    "Mounts": [
            {
                "Source": "/home/docker/tomcat/webapps",
                "Destination": "/usr/local/tomcat/webapps",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ]
    "Ports": {
                "8080/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "8888"
                    }
                ]
            }

### 利用卷进行数据备份
    # 备份

    # 通过ubuntu镜像新建一个容器，引用tomcat1容器的数据卷，并创建一个数据卷映射
    # 在用户目录$HOME下创建backup文件夹

    $ cd ~ && mkdir backup
    $ docker pull ubuntu:15.10
    # 使用ubuntu镜像作为专门用于挂载数据卷的容器
    $ docker run --volumes-from=tomcat1 -v $HOME/backup:/backup \
      ubuntu:15.10 tar zcf /backup/www.tar.gz /usr/local/tomcat/webapps/

    # 将宿主机$HOME目录下的backup文件映射到容器的/backup目录，通过tar命令将/usr/local/tomcat/web	apps目录压缩打包到/backup数据卷中，
    # 该数据卷又映射到了宿主机$HOME/backup，故容器中的数    	据就保存到了宿主机中，文件名为：www.tar.gz

---

    # 恢复

    # 创建容器tomcat3，并创建数据卷
    $ docker run -d -p 8889:8080 -v /usr/local/tomcat/webapps --name tomcat3 tomcat
    # -v参数：在容器中创建数据卷/usr/local/tomcat/webapps
    # 基于ubuntu镜像创建新容器来关联到宿主机目录$HOME/backup 并将数据解压到数据卷中
    # 新容器与tomcat3容器共享数据卷
    $ docker run --volumes-from=tomcat3 -v $HOME/backup:/backup ubuntu:15.10 tar -zxf /backup/www.tar.gz
    # 测试： localhost:8889/project

>注：本文不限转载，但转载时请注明出处
