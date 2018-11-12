---
title: docker常用命令
date: 2018-11-02 14:18:32
tags:
    - docker
    - 常用命令
---

* 查看docker版本信息  
docker version 

* 查看当前的镜像列表  
docker images 

* 查看当前运行的容器  
docker ps  
docker ps －a 

* 启动docker 容器  
docker run -t -i ubuntu:14.04 /bin/bash   
docker run -idt ubuntu:14.04 （后台启动）

<!-- more -->

* 加入到后台开启页面   
docker attach containerID 

* 将文件复制到docker中去  
docker cp 文件路径 containerID:目标文件路径  
docker cp containerID:文件路径 目标文件路径

* 提交生成好的docker为新的镜像  
docker commit -m “Added json gem” -a “Docker Newbee” 0b2616b0e5a8 ouruser/sinatra:v2 4f177bd27a9ff0f6dc2a830403925b5360bfe0b93d476f7fc3231110e7f71b1c 

* 删除容器  
1. 删除单个容器  
docker rm container_name 
2. 删除所有未运行的容器  
docker rm $(docker ps -a -q) 

* 删除镜像  
docker rmi ubuntu:v1  

* 修改镜像标签名  
docker tag old-image[:old-tag] new-image[:new-tag]  

* 退出docker  
exit 

* 退出后再进入 
1. 先查到已经退出的容器  
docker ps -a  
2. 找到要启动的容器名称，后通过start启动  
docker start goofy_almeida（容器名称或容器id）  
3. 再通过attach进入到命令行  
docker attach goofy_almeida（容器名称或容器id）  
或使用exec方式进入命令行，这种方式 在内部使用exit的时候，会继续在后台运行 
docker exec -it [CONTAINER_NAME or CONTAINER_ID] /bin/bash 

* 启动spring boot web应用(需要先安装好jdk环境)  
java -jar sboot.jar  
启动后理论上就可以访问了，但是由于外部端口不能直接访问，所以需要宿主机将外部端口与docker绑定后才能访问 
1. 提交准备好的环境  
docker commit 57c312bbaad1 fanhq/javaweb:0.1  
2. 通过绑定端口的方式启动doker  
docker run -t -i -p 58080:8080 –name springboot(自定义别名)  fanhq/javaweb:0.1 /bin/bash  
3. 进入doker后启动项目  
java -jar sboot.jar 
4. 访问"http://192.168.65.132:58080/"（ip为虚拟机的Ip）
* 在docker中部署环境，以springboot的应用为例
1. 首先要安装vi指令或vim指令用于编辑环境变量文件，也可以不用安装，可将docker容器中的文件复制到宿主机上修改  
vi指令安装  
apt-get install vi  
apt-get update 
2. 安装jdk 
官网下载linux－x64.tar的jdk包  
放到虚拟机共享目录中  
复制到doker容器虚拟文件目录中去  
docker cp linux－x64.tar containerID:/opt/linux－x64.tar  
解压tar包  
tar -xvf linux－x64.tar  
将包重新命名为jdk  
mv linux－x64 jdk  
设置环境变量  
vi ~/.bashrc(~表示root目录下的文件，可用命令验证目录文件)  
在文件末尾添加  
export JAVA_HOME=/opt/jdk  
export PATH=PATH:JAVA_HOME/bin  
执行source命令让source命令生效  
source ~/.bashrc  
测试jdk版本  
java -version  
3. 准备springboot 包  
写一个简单的springboot项目并打包为胖jar  
4. 将文件复制到doker中去  
5. 运行jar包  
java -jar sboot.jar  
运行不报错准备工作基本已完成，然后将容器重新打包为镜像 
6. 容器提交为镜像  
docker commit 57c312bbaad1 fanhq/javaweb:0.1 
7. 绑定端口运行容器  
第5步启动的项目外部无法访问到，因为端口未开放，所以需要开放端口绑定运行一个容器  
docker run -t -i -p 58080:8080 fanhq/javaweb:0.1 /bin/bash  
docker run -d -p 58080:8080 fanhq/javaweb:0.1 /bin/bash 
8. 进入容器后重新运行第5步的启动项目，然后通过外部端口访问 
http://192.168.65.132:58080/   
至此，一个手动版的doker部署就搞定了，下一步是自动化发布启动doker。

* Dockerfile 示例
``` shell
# 指定基础镜像
FROM sameersbn/ubuntu:14.04.20161014

# 维护者信息
MAINTAINER sameer@damagehead.com

# 设置环境
ENV RTMP_VERSION=1.1.10 \
NPS_VERSION=1.11.33.4 \
LIBAV_VERSION=11.8 \
NGINX_VERSION=1.10.1 \
NGINX_USER=www-data \
NGINX_SITECONF_DIR=/etc/nginx/sites-enabled \
NGINX_LOG_DIR=/var/log/nginx \
NGINX_TEMP_DIR=/var/lib/nginx \
NGINX_SETUP_DIR=/var/cache/nginx

# 设置构建时变量，镜像建立完成后就失效
ARG BUILD_LIBAV=false
ARG WITH_DEBUG=false
ARG WITH_PAGESPEED=true
ARG WITH_RTMP=true

# 复制本地文件到容器目录中
COPY setup/ ${NGINX_SETUP_DIR}/
RUN bash ${NGINX_SETUP_DIR}/install.sh

# 复制本地配置文件到容器目录中
COPY nginx.conf /etc/nginx/nginx.conf
COPY entrypoint.sh /sbin/entrypoint.sh

# 运行指令
RUN chmod 755 /sbin/entrypoint.sh

# 允许指定的端口
EXPOSE 80/tcp 443/tcp 1935/tcp

# 指定网站目录挂载点
VOLUME ["${NGINX_SITECONF_DIR}"]

ENTRYPOINT ["/sbin/entrypoint.sh"]
CMD ["/usr/sbin/nginx"]
```
* Docker Compose  
    - 示例
    ``` shell
    version: '2'
    services:
            db:
                    image: mysql:5.7
                    volumes: 
                            - "./.data/db:/var/lib/mysql"
                    restart: always
                    environment:
                            MYSQL_ROOT_PASSWORD: wordpress
                            MYSQL_DATABASE: wordpress
                            MYSQL_USER: wordpress
                            MYSQL_PASSWORD: wordpress

            wordpress:
                    depends_on: 
                            - db
                    image: wordpress:latest
                    links:
                            - db
                    ports:
                            - "80:80"
                    restart: always
                    environment:
                            WORDPRESS_DB_HOST: db:3306
                            WORDPRESS_DB_PASSWORD: wordpress
    ```
    - 命令
    ``` shell
    #查看帮助
    docker-compose -h

    # -f  指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定。
    docker-compose -f docker-compose.yml up -d 

    #启动所有容器，-d 将会在后台启动并运行所有的容器
    #默认解析当前目录的docker-compose.yml文件
    docker-compose up -d

    #停用移除所有容器以及网络相关
    docker-compose down

    #查看服务容器的输出
    docker-compose logs

    #列出项目中目前的所有容器
    docker-compose ps

    #构建（重新构建）项目中的服务容器。服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。可以随时在项目目录下运行 docker-compose build 来重新构建服务
    docker-compose build

    #拉取服务依赖的镜像
    docker-compose pull

    #重启项目中的服务
    docker-compose restart

    #删除所有（停止状态的）服务容器。推荐先执行 docker-compose stop 命令来停止容器。
    docker-compose rm 

    #在指定服务上执行一个命令。
    docker-compose run ubuntu ping docker.com

    #设置指定服务运行的容器个数。通过 service=num 的参数来设置数量
    docker-compose scale web=3 db=2

    #启动已经存在的服务容器。
    docker-compose start

    #停止已经处于运行状态的容器，但不删除它。通过 docker-compose start 可以再次启动这些容器。
    docker-compose stop


    ```
