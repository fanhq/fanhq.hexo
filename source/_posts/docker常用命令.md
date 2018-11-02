---
title: docker常用命令
date: 2018-11-02 14:18:32
tags:
    -docker
    -常用命令
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

4. 访问http://192.168.65.132:58080/（ip为虚拟机的Ip）

* 在docker中安装开发环境 
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
使用dockerfile来创建镜像，建议这种方式生成容器，便于管理

* 文档内容 
FROM bootrun  
ENV JAVA_HOME /opt/jdk   
ENV PATH JAVAHOME/bin:PATH  
ENV CLASSPATH .:$JAVA_HOME/lib  
#COPY /Users/wangchun/Desktop/sboot.jar /sboot.jar 
EXPOSE 8080  
ENTRYPOINT java -jar /sboot.jar && /bin/bash 

* 设置 
docker build -t=”redstarofsleep/javaweb” .