# Docker-Compose的使用

```
layout:     post
title:      docker-compose介绍
subtitle:   docker-compose如何使用
date:       2020-01-10
author:     BY
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - docker

```

Docker是一个开源的容器引擎，它有助于更快地交付应用。方便快捷已经是 Docker的最大优势，过去需要用数天乃至数周的任务，在Docker容器的处理下，只需要数秒就能完成。

# 架构



![img](https:////upload-images.jianshu.io/upload_images/3264969-fbafaa59ab75744e.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Docker架构图

- Docker daemon（ Docker守护进程）：Docker daemon是一个运行在宿主机（ DOCKER-HOST）的后台进程。可通过 Docker客户端与之通信。
- Client（ Docker客户端）：Docker客户端是 Docker的用户界面，它可以接受用户命令和配置标识，并与 Docker daemon通信。图中， docker build等都是 Docker的相关命令。
- Images（ Docker镜像）：Docker镜像是一个只读模板，它包含创建 Docker容器的说明。它和系统安装光盘有点像，使用系统安装光盘可以安装系统，同理，使用Docker镜像可以运行 Docker镜像中的程序。
- Container（容器）：容器是镜像的可运行实例。镜像和容器的关系有点类似于面向对象中，类和对象的关系。可通过 Docker API或者 CLI命令来启停、移动、删除容器。
- Registry：Docker Registry是一个集中存储与分发镜像的服务。构建完 Docker镜像后，就可在当前宿主机上运行。但如果想要在其他机器上运行这个镜像，就需要手动复制。此时可借助Docker Registry来避免镜像的手动复制。一个 Docker Registry可包含多个 Docker仓库，每个仓库可包含多个镜像标签，每个标签对应一个 Docker镜像。这跟 Maven的仓库有点类似，如果把 Docker Registry比作Maven仓库的话，那么 Docker仓库就可理解为某jar包的路径，而镜像标签则可理解为jar包的版本号。Docker Registry可分为公有Docker Registry和私有Docker Registry。 最常用的DockerRegistry莫过于官网的Docker Hub， 这也是默认的Docker Registry。 Docker Hub上存放着大量优秀的镜像， 我们可使用Docker命令下载并使用。

# 安装

按照菜鸟的步骤，使用yum安装即可。

# 常用命令

### 镜像相关

- docker search java：在Docker Hub（或阿里镜像）仓库中搜索关键字（如java）的镜像
- docker pull java:8：从仓库中下载镜像，若要指定版本，则要在冒号后指定
- docker images：列出已经下载的镜像
- docker rmi java：删除本地镜像
- docker build：构建镜像



### 容器相关

- docker run -d -p 91:80 nginx ：在后台运行nginx，若没有镜像则先下载，并将容器的80端口映射为宿主机的91端口。 
  - -d：后台运行
  - -P：随机端口映射
  - -p：指定端口映射
  - -net：网络模式
- docker ps：列出运行中的容器
- docker ps -a ：列出所有的容器
- docker stop 容器id：停止容器
- docker kill 容器id：强制停止容器
- docker start 容器id：启动已停止的容器
- docker inspect 容器id：查看容器的所有信息
- docker container logs 容器id：查看容器日志
- docker top 容器id：查看容器里的进程
- docker exec -it 容器id /bin/bash：进入容器
- exit：退出容器
- docker rm 容器id：删除已停止的容器
- docker rm -f 容器id：删除正在运行的容器

### 所有命令

- docker
- docker COMMAND --help

# 构建镜像

1. 确定镜像模板：如java、nginx
2. 新建Dockerfile文件
3. 使用Dockerfile的指令完善Dockerfile的内容
4. 在Dockerfile文件的所在路径执行`docker build -t imageName:tag .`，-t指定镜像名称，末尾的点标识Dockerfile文件的路径
5. 执行`docker run -d -p 92:80 imageName:tag`即可

常用指令如下图，直白用法点[我](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fniloay%2Fp%2F6261784.html)，官方介绍点击[我](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.docker.com%2Fv17.09%2Fengine%2Freference%2Fbuilder%2F)




![img](https:////upload-images.jianshu.io/upload_images/3264969-4a084303f9049fa8.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/563/format/webp)

个性化指令解释



备注：RUN命令在 image 文件的构建阶段执行，执行结果都会打包进入 image 文件；CMD命令则是在容器启动后执行。另外，一个 Dockerfile 可以包含多个RUN命令，但是只能有一个CMD命令。注意，指定了CMD命令以后，docker container run命令就不能附加命令了，否则它会覆盖CMD命令。

# Docker Compose

[Docker Compose](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fdocker%2Fcompose)是 docker 提供的一个命令行工具，用来定义和运行由多个容器组成的应用。使用 compose，我们可以通过 YAML 文件声明式的定义应用程序的各个服务，并由单个命令完成应用的创建和启动

