---
title: Docker笔记
index_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Docker笔记.png
banner_img: https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Docker笔记.png
layout: post
categories: [Notebook]
tags: [Docker]
---

一个开源的应用容器引擎

<!-- more -->

## 1 概述

### 1.1 容器生态系统

Docker现在几乎是容器的代名词。确实，是Docker将容器技术发扬光大。同时，大家也需要知道围绕Docker还有一个生态系统。Docker是这个生态系统的基石，但完善的生态系统才是保障Docker以及容器技术能够真正健康发展的决定因素。

容器生态系统：

- 容器核心技术

  容器核心技术是指能够让Container在host上运行起来的那些技术。

  - 容器规范

    容器不光是Docker，还有其他容器，比如CoreOS的rkt。为了保证容器生态的健康发展，保证不同容器之间能够兼容，包含Docker、CoreOS、Google在内的若干公司共同成立了一个叫Open Container Initiative（OCI）的组织，其目的是制定开放的容器规范。目前OCI发布了两个规范：

    - runtime spec
    - image format spec

  - 容器runtime

    runtime是容器真正运行的地方。runtime需要跟操作系统kernel紧密协作，为容器提供运行环境。如果大家用过Java，可以这样来理解runtime与容器的关系：Java程序就好比是容器，JVM则好比是runtime, JVM为Java程序提供运行环境。同样的道理，容器只有在runtime中才能运行。lxc、runc和rkt是目前主流的三种容器runtime。

    lxc是Linux上老牌的容器runtime。Docker最初也是用lxc作为runtime。runc是Docker自己开发的容器runtime，符合oci规范，也是现在Docker的默认runtime。rkt是CoreOS开发的容器runtime，符合OCI规范，因而能够运行Docker的容器。

    - lxc
    - runc
    - rkt

    

  - 容器管理工具

    容器管理工具对内与runtime交互，对外为用户提供interface，比如CLI。这就好比除了JVM，还得提供Java命令让用户能够启停应用。

    runc的管理工具是docker engine。docker engine包含后台deamon和cli两个部分。我们通常提到Docker，一般就是指的docker engine。

    - lxd

    - docker engine
      - deamon

      - cli

    - rkt cli

  - 容器定义工具

    容器定义工具允许用户定义容器的内容和属性，这样容器就能够被保存、共享和重建。

    docker image是Docker容器的模板，runtime依据docker image创建容器。dockerfile是包含若干命令的文本文件，可以通过这些命令创建出docker image。ACI(App Container Image)与docker image类似，只不过它是由CoreOS开发的rkt容器的image格式。

    - docker image
    - dockerfile
    - ACI (App Container Image)

  - Registries

    容器是通过image创建的，需要有一个仓库来统一存放image，这个仓库就叫做Registry。

    - Docker Registry
    - Docker Hub
    - Quay.io

  - 容器OS

    由于有容器runtime，几乎所有的Linux、MAC OS和Windows都可以运行容器，但这并没有妨碍容器OS的问世。容器OS是专门运行容器的操作系统。与常规OS相比，容器OS通常体积更小，启动更快。因为是为容器定制的OS，通常它们运行容器的效率会更高。目前已经存在不少容器OS, CoreOS、Atomic和Ubuntu Core是其中的杰出代表。

    - CoreOS
    - Atomic
    - Ubuntu Core

- 容器平台技术

  容器核心技术使得容器能够在单个host上运行，而容器平台技术能够让容器作为集群在分布式环境中运行。

  - 容器编排引擎

    基于容器的应用一般会采用微服务架构。在这种架构下，应用被划分为不同的组件，并以服务的形式运行在各自的容器中，通过API对外提供服务。为了保证应用的高可用，每个组件都可能会运行多个相同的容器。这些容器会组成集群，集群中的容器会根据业务需要被动态地创建、迁移和销毁。

    所谓编排（orchestration），通常包括容器管理、调度、集群定义和服务发现等。通过容器编排引擎，容器被有机地组合成微服务应用，实现业务需求。

    - docker swarm
    - kubernetes
    - mesos+marathon

  - 容器管理平台

    容器管理平台是架构在容器编排引擎之上的一个更为通用的平台。通常容器管理平台能够支持多种编排引擎，抽象了编排引擎的底层实现细节，为用户提供更方便的功能，比如application catalog和一键应用部署等。

    - Rancher

    - ContainerShip

  - 基于容器的PaaS

    基于容器的PaaS为微服务应用开发人员和公司提供了开发、部署和管理应用的平台，使用户不必关心底层基础设施而专注于应用的开发。

    - Deis
    - Flynn
    - Dokku

- 容器支持技术

  - 容器网络

    容器的出现使网络拓扑变得更加动态和复杂。用户需要专门的解决方案来管理容器与容器、容器与其他实体之间的连通性和隔离性。docker network是Docker原生的网络解决方案。

    - docker network
    - flannel
    - weave
    - calico

  - 服务发现

    动态变化是微服务应用的一大特点。当负载增加时，集群会自动创建新的容器；负载减小，多余的容器会被销毁。容器也会根据host的资源使用情况在不同host中迁移，容器的IP和端口也会随之发生变化。

    在这种动态的环境下，必须要有一种机制让client能够知道如何访问容器提供的服务。这就是服务发现技术要完成的工作。

    服务发现会保存容器集群中所有微服务最新的信息，比如IP和端口，并对外提供API，提供服务查询功能。

    - etcd
    - consul
    - zookeeper

  - 监控

    docker ps/top/stats是Docker原生的命令行监控工具。除了命令行，Docker也提供了stats API，用户可以通过HTTP请求获取容器的状态信息。

    - docker ps/top/stats
    - docker stats API
    - sysdig
    - cAdvisor/Heapster
    - Weave Scope

  - 数据管理

    容器经常会在不同的host之间迁移，如何保证持久化数据也能够动态迁移，是Rex-Ray这类数据管理工具提供的能力

    - REX-Ray

  - 日志管理

    日志为问题排查和事件管理提供了重要依据。

    docker logs是Docker原生的日志工具。而logspout对日志提供了路由功能，它可以收集不同容器的日志并转发给其他工具进行后处理。

    - docker logs
    - logspout

  - 安全性

    OpenSCAP能够对容器镜像进行扫描，发现潜在的漏洞。

    - OpenSCAP

### 1.2 容器核心知识

容器是一种轻量级、可移植、自包含的软件打包技术，使应用程序可以在几乎任何地方以相同的方式运行。开发人员在自己笔记本上创建并测试好的容器，无须任何修改就能够在生产系统的虚拟机、物理服务器或公有云主机上运行。

容器由两部分组成：

（1）应用程序本身；

（2）依赖：比如应用程序需要的库或其他软件容器在Host操作系统的用户空间中运行，与操作系统的其他进程隔离。这一点显著区别于的虚拟机。

传统的虚拟化技术，比如VMWare、KVM、Xen，目标是创建完整的虚拟机。为了运行应用，除了部署应用本身及其依赖（通常几十MB），还得安装整个操作系统（几十GB）。由于所有的容器共享同一个Host OS，这使得容器在体积上要比虚拟机小很多。另外，启动容器不需要启动整个操作系统，所以容器部署和启动速度更快、开销更小，也更容易迁移。

![VM与Container](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-VM%E4%B8%8EContainer.jpg)

Docker将集装箱思想运用到软件打包上，为代码提供了一个基于容器的标准化运输系统。Docker可以将任何应用及其依赖打包成一个轻量级、可移植、自包含的容器。

只需要配置好标准的runtime环境，服务器就可以运行任何容器。这使得运维人员的工作变得更高效、一致和可重复。容器消除了开发、测试、生产环境的不一致性。

Docker的核心组件包括：

- Docker客户端：Client

- Docker服务器：Docker daemon

- Docker镜像：Image

- Registry

- Docker容器：Container

![Docker架构](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Docker%E6%9E%B6%E6%9E%84.jpg)

Docker采用的是Client/Server架构。客户端向服务器发送请求，服务器负责构建、运行和分发容器。客户端和服务器可以运行在同一个Host上，客户端也可以通过socket或REST API与远程的服务器通信。

## 2 环境

容器需要管理工具、runtime和操作系统，我们的选择如下：

（1）管理工具：Docker Engine。因为Docker最流行使用最广泛。

（2）runtime :runc。Docker的默认runtime。

（3）操作系统：Ubuntu。虽然存在诸如CoreOS的容器OS，因考虑到我们目前处于初学阶段，选择大家熟悉的操作系统更为合适。等具备了扎实的容器基础知识后，再使用容器OS会更有利。

Docker分为开源免费的CE（Community Edition）版本和收费的EE（Enterprise Edition）版本。

国内 daocloud 一键安装命令：

```bash
curl -sSL https://get.daocloud.io/docker | sh
```

## 3 应用

### 3.1 快速入门

运行容器：

```bash
docker run -d -p 80:80 httpd
```

访问http://localhost

上述命令执行过程如下：

（1）Docker客户端执行docker run命令。启动httpd容器，并将容器的80端口映射到host的80端口。

（2）Docker daemon发现本地没有httpd镜像。

（3）daemon从Docker Hub下载镜像。

（4）下载完成，镜像httpd被保存到本地。

（5）Docker daemon启动容器。

### 3.2 使用镜像

可将Docker镜像看成只读模板，通过它可以创建Docker容器。对于应用软件，镜像是软件生命周期的构建和打包阶段，而容器则是启动和运行阶段。

镜像名为REPOSITORY，完整格式为：[registry-host]:[port]/[username]/xxx。只有Docker Hub上的镜像可以省略registry-host:[port]。不可以用 IMAGE ID代替。

```bash
# 拉取镜像
docker pull [镜像名]
# 查看镜像
docker images [镜像名]
# 运行容器
docker run [镜像名]
# 显示运行容器
docker ps
docker container ls
# 删除Docker host中的镜像
docker rmi [镜像名]
# 搜索Docker Hub中的镜像
docker search [镜像名]
```

#### 3.2.1 Dockerfile介绍

Dockerfile是镜像的描述文件，定义了如何构建Docker镜像。

同样创建一个新镜像

1. 使用docker commit:

```bash
# -it参数的作用是以交互模式进入容器，并打开终端。
docker run -it ubuntu
# 安装vi
apt-get install -y vim
# 保存为新镜像
docker commit [容器名] [新镜像命名]
```

2. 使用Dockerfile:

```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y vim
```

```bash
# -t将新镜像命名为ubuntu-with-vi-dockerfile，命令末尾的．指明build context为当前目录。Docker默认会从build context中查找Dockerfile文件，我们也可以通过-f参数指定Dockerfile的位置。
docker build -t [新镜像命名] .
# 显示镜像的构建历史，也就是Dockerfile的执行过程。
docker history [镜像名]
```

Docker会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无须重新创建。如果我们希望在构建镜像时不使用缓存，可以在docker build命令中加上--no-cache参数。

**通过Dockerfile构建镜像的过程**：

1. 从base镜像运行一个容器。

2. 执行一条指令，对容器做修改。

3. 执行类似docker commit的操作，生成一个新的镜像层。

4. Docker再基于刚刚提交的镜像运行一个新容器。

5. 重复2～4步，直到Dockerfile中的所有指令执行完毕。

从这个过程可以看出，如果Dockerfile由于某种原因执行到某个指令失败了，我们也将能够得到前一个指令成功执行构建出的镜像，这对调试Dockerfile非常有帮助。我们可以运行最新的这个镜像定位指令失败的原因。

**Dockerfile语法**：

- FROM 指定base镜像。

- MAINTAINER 设置镜像的作者，可以是任意字符串。

- COPY 将文件从build context复制到镜像。COPY支持两种形式： COPY src dest与COPY ["src", "dest"]。注意：src只能指定build context中的文件或目录。

- ADD 与COPY类似，从build context复制文件到镜像。不同的是，如果src是归档文件（tar、zip、tgz、xz等），文件会被自动解压到dest。

- ENV 设置环境变量，环境变量可被后面的指令使用。例如：

  ```dockerfile
  ENV MY_VERSION 1.3 
  RUN apt-get install -y mypackage=$MY_VERSION
  ```

- EXPOSE 指定容器中的进程会监听某个端口，Docker可以将该端口暴露出来。我们会在容器网络部分详细讨论。

- VOLUME 将文件或目录声明为volume。我们会在容器存储部分详细讨论。

- WORKDIR 为后面的RUN、CMD、ENTRYPOINT、ADD或COPY指令设置镜像中的当前工作目录。

- RUN 在容器中运行指定的命令。

- CMD 容器启动时运行指定的命令。Dockerfile中可以有多个CMD指令，但只有最后一个生效。CMD可以被docker run之后的参数替换。

- ENTRYPOINT 设置容器启动时运行的命令。Dockerfile中可以有多个ENTRYPOINT指令，但只有最后一个生效。CMD或docker run之后的参数会被当作参数传递给ENTRYPOINT。

#### 3.2.2 镜像示例

hello-world是Docker官方提供的一个镜像，通常用来验证Docker是否安装成功。它还不到还不到2KB。hello-world的Dockerfile内容：

```dockerfile
FROM scratch
COPY hello /
CMD ["/hello"]
```

1. FROM scratch 镜像是从白手起家，从0开始构建。

2. COPY hello / 将文件“hello”复制到镜像的根目录。

3. CMD ["/hello"] 容器启动时，执行/hello。

base镜像有两层含义：（1）不依赖其他镜像，从scratch构建；（2）其他镜像可以以之为基础进行扩展。能称作base镜像的通常都是各种Linux发行版的Docker镜像，比如Ubuntu、Debian、CentOS等。一个CentOS镜像只有200MB。

![Linux操作系统组成](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-Linux%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%BB%84%E6%88%90.jpg)

Linux操作系统由内核空间和用户空间组成。内核空间是kernel, Linux刚启动时会加载bootfs文件系统，之后bootfs会被卸载掉。用户空间的文件系统是rootfs，包含我们熟悉的 /dev、/proc、/bin等目录。对于base镜像来说，底层直接用Host的kernel，自己只需要提供rootfs就行了。而对于一个精简的OS, rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了。相比其他Linux发行版，CentOS的rootfs已经算臃肿的了，alpine还不到10MB。

CentOS镜像的Dockerfile的内容：

```dockerfile
FROM scratch
ADD centos-7-docker.tar.xz /
CMD ["/bin/bash"]
```

不同Linux发行版的区别主要就是rootfs。比如Ubuntu 14.04使用upstart管理服务，apt管理软件包；而CentOS 7使用systemd和yum。这些都是用户空间上的区别，Linux kernel差别不大。所以Docker可以同时支持多种Linux镜像，模拟出多种操作系统环境。

base镜像只是在用户空间与发行版一致，**kernel版本与发行版是不同的**。所有容器都共用host的kernel，在容器中没办法对kernel升级。如果容器对kernel版本有要求（比如应用只能在某个kernel版本下运行），则不建议用容器**，这种场景虚拟机可能更合适**。

Docker Hub中99%的镜像都是通过在base镜像中安装和配置需要的软件构建出来的。比如我们现在构建一个新的镜像：

```dockerfile
FROM debian
RUN apt-get install emacs
RUN apt-get install apaches
CMD ["/bin/bash"]
```

![构建过程](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B.jpg)

可以看到，新镜像是从base镜像一层一层叠加生成的。每安装一个软件，就在现有镜像的基础上增加一层。

#### 3.2.3 容器Copy-on-Write特性

当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。

![容器层](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%AE%B9%E5%99%A8%E5%B1%82.jpg)

所有对容器的改动，无论添加、删除，还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的所有镜像层都是只读的。镜像层数量可能会很多，所有镜像层会联合在一起组成一个统一的文件系统。如果不同层中有一个相同路径的文件，比如/a，上层的 /a会覆盖下层的 /a，也就是说用户只能访问到上层中的文件 /a。**在容器层中，用户看到的是一个叠加之后的文件系统。**

（1）添加文件。在容器中创建文件时，新文件被添加到容器层中。

（2）读取文件。在容器中读取某个文件时，Docker会从上往下依次在各镜像层中查找此文件。一旦找到，打开并读入内存。

（3）修改文件。在容器中修改已存在的文件时，Docker会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。

（4）删除文件。在容器中删除文件时，Docker也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。

只有当需要修改时才复制一份数据，这种特性被称作Copy-on-Write。可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。容器层记录对镜像的修改，所有镜像层都是只读的，不会被容器修改，**所以镜像可以被多个容器共享**。

#### 3.2.4 RUN vs CMD vs ENTRYPOINT

（1）RUN：执行命令并创建新的镜像层，RUN经常用于安装软件包。

（2）CMD：设置容器启动后默认执行的命令及其参数，但CMD能够被docker run后面跟的命令行参数替换。

（3）ENTRYPOINT：配置容器启动时运行的命令。ENTRYPOINT看上去与CMD很像，它们都可以指定要执行的命令及其参数。不同的地方在于ENTRYPOINT不会被忽略，一定会被执行，即使运行docker run时指定了其他命令。

我们可用两种方式指定RUN、CMD和ENTRYPOINT要运行的命令：Shell格式和Exec格式，二者在使用上有细微的区别。

Shell格式：

```dockerfile
RUN apt-get install python3    
CMD echo "Hello world"    
ENTRYPOINT echo "Hello world"
```

Exec格式：

```dockerfile
 RUN ["apt-get", "install", "python3"]   
 CMD ["/bin/echo", "Hello world"]    
 ENTRYPOINT ["/bin/echo", "Hello world"]
```

CMD和ENTRYPOINT推荐使用Exec格式，因为指令可读性更强，更容易理解。RUN则两种格式都可以。

实践经验：

（1）使用RUN指令安装应用和软件包，构建镜像。

（2）如果Docker镜像的用途是运行应用程序或服务，比如运行一个MySQL，应该优先使用Exec格式的ENTRYPOINT指令。CMD可为ENTRYPOINT提供额外的默认参数，同时可利用docker run命令行替换默认参数。

（3）如果想为容器设置默认的启动命令，可使用CMD指令。用户可在docker run命令行中替换此默认命令。
#### 3.2.5 分发镜像

如果执行docker build时没有指定tag，会使用默认值latest。tag常用于描述镜像的版本信息。

```bash
docker tag myimage-v1.9.2 myimage:1 
docker tag myimage-v1.9.2 myimage:1.9 
docker tag myimage-v1.9.2 myimage:1.9.2 
docker tag myimage-v1.9.2 myimage:latest
```

打tag之后会生成新的镜像引用。用tag删除仅删除引用（只有当最后一个tag被删除时，镜像才被真正删除），用镜像ID删除会删除镜像。

Docker Hub为了区分不同用户的同名镜像，镜像的registry中要包含用户名，完整格式为：[username]/xxx:tag。

```bash
# 登陆Docker Host
docker login -u [用户名]
# 通过docker push将镜像上传到Docker Hub
docker push [镜像名]
```

如果想上传同一repository中所有镜像，省略tag部分就可以了。

### 3.4 使用容器

**容器有三种名字**：

CONTAINER ID是容器的“短ID”，启动容器时返回的则是“长ID”。短ID是长ID的前12个字符。NAMES字段显示容器的名字，在启动容器时可以通过 --name参数显式地为容器命名，如果不指定，docker会自动为容器分配名字。

```bash
# 运行容器
docker run [容器名]
# 显示容器列表,-a显示全部
docker ps
docker container ls
# 停止,发送SIGTERM信号
docker stop [容器名]
# 停止，发送SIGKILL信号
docker kill [容器名]
# 启动停止状态的容器，保留容器的第一次启动时的所有参数
docker start [容器名]
# 依次执行docker stop和docker start
docker restart [容器名]
# docker run 添加参数 --restart=always意味着无论容器因何种原因退出（包括正常退出），都立即重启；该参数的形式还可以是 --restart=on-failure:3，意思是如果启动进程退出代码非0，则重启容器，最多重启3次。
# 处于暂停状态的容器不会占用CPU资源，直到通过docker unpause恢复运行
docker pause/unpause [容器名]
# 如果确认不会再重启此类容器，可以通过docker rm删除，一次可以指定多个
docker rm [容器名]
# 如果希望批量删除所有已经退出的容器，可以执行如下命令
docker rm -v $(docker ps -aq -f status=exited)
```

可用三种方式指定容器启动时执行的命令：

（1）CMD指令。

（2）ENTRYPOINT指令。

（3）在docker run命令行中指定。

**容器分类**：

**服务类容器**以daemon的形式运行，对外提供服务，比如Web Server、数据库等。通过 -d以后台方式启动这类容器是非常合适的。如果要排查问题，可以通过exec -it进入容器。**工具类容器**通常能给我们提供一个临时的工作环境，通常以run -it方式运行。

#### 3.4.1 attach VS exec

```bash
# attach到容器启动命令的终端
docker attach [容器名]
# 通过docker exec进入相同的容器
docker exec -it [容器名] bash|sh
```

（1）attach直接进入容器启动命令的终端，不会启动新的进程。（2）exec则是在容器中打开新的终端，并且可以启动新的进程。（3）如果想直接在终端中查看启动命令的输出，用attach；其他情况使用exec。当然，如果只是为了查看启动命令的输出，可以使用docker logs命令。

如果要退出，输入exit或使用ctrl+D。使用attach命令后从 stdin 中 exit，会导致容器的停止；使用exec命令后从这个 stdin 中 exit，不会导致容器的停止。

#### 3.4.2 容器状态机

![容器状态机](https://cdn.jsdelivr.net/gh/bit704/blog-image-bed@main/image/2022-09-18-%E5%AE%B9%E5%99%A8%E7%8A%B6%E6%80%81%E6%9C%BA.jpg)

可以先创建容器，稍后再启动。docker start将以后台方式启动容器。docker run命令实际上是docker create和docker start的组合。

退出包括正常退出或者非正常退出。这里举了两个例子：启动进程正常退出或发生OOM，此时Docker会根据 --restart的策略判断是否需要重启容器。但如果容器是因为执行docker stop或docker kill退出，则不会自动重启。

#### 3.4.3 资源限制

与操作系统类似，容器可使用的内存包括两部分：物理内存和swap。Docker通过下面两组参数来控制容器内存的使用量。

（1）-m 或 --memory：设置内存的使用限额，例如100MB,2GB。

（2）--memory-swap：设置内存+swap的使用限额。

（3）-c 或 --cpu-shares：设置容器使用CPU的权重。某个容器最终能分配到的CPU资源取决于它的cpu share占所有容器cpu share总和的比例。

（4）--cpu：设置工作线程的数量。

（5）--blkio-weight：设置容器读写磁盘的权重。

（6）bps是byte per second，每秒读写的数据量。iops是io per second，每秒IO的次数。

- --device-read-bps：限制读某个设备的bps。
-  --device-write-bps：限制写某个设备的bps。
- --device-read-iops：限制读某个设备的iops。
- --device-write-iops：限制写某个设备的iops。

```bash
# 以下命令含义是允许该容器最多使用200MB的内存和100MB的swap。默认情况下，两组参数为 -1，即对容器内存和swap的使用没有限制。如果在启动容器时只指定 -m而不指定 --memory-swap，那么--memory-swap默认为 -m的两倍
docker run -m 200M --memory-swap=300M ubuntu
# 默认设置下，所有容器可以平等地使用host CPU资源并且没有限制。如果不指定，默认值为1024。
docker run --name "container_A" -c 1024 ubuntu --cpu 1
# 限制容器写 /dev/sda的速率为30 MB/s
docker run -it --device-write-bps /dev/sda:30MB ubuntu
```

#### 3.4.4 底层技术

cgroup和namespace是最重要的两种技术。cgroup实现资源限额，namespace实现资源隔离。

cgroup全称Control Group。Linux操作系统通过cgroup可以设置进程使用CPU、内存和IO资源的限额。 --cpu-shares、-m、--device-write-bps实际上就是在配置cgroup。在 /sys/fs/cgroup/cpu/docker目录中，Linux会为每个容器创建一个cgroup目录，以容器长ID命名。

namespace管理着host中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。换句话说，namespace实现了容器间资源的隔离。Linux使用了6种namespace，分别对应6种资源：

- Mount

  Mount namespace让容器看上去拥有整个文件系统。容器有自己的/目录，可以执行mount和umount命令。

- UTS

  UTS namespace让容器有自己的hostname。默认情况下，容器的hostname是它的短ID，可以通过 -h或--hostname参数设置

- IPC

  IPC namespace让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与host和其他容器的IPC混在一起。

- PID

  进程的PID不同于host中对应进程的PID，容器中PID=1的进程当然也不是host的init进程。也就是说：容器拥有自己独立的一套PID，这就是PID namespace提供的功能。

- Network

  Network namespace让容器拥有自己独立的网卡、IP、路由等资源。

- User

  User namespace让容器能够管理自己的用户，host不能看到容器中创建的用户

## 参考

《每天五分钟玩转Docker容器技术》--CloudMan  前四章

[Docker教程_菜鸟](https://www.runoob.com/docker/docker-tutorial.html)

