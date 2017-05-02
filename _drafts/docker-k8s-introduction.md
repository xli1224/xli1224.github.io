---
layout: post
title: "Docker和Kubernetes介绍"
auther: "Li Xiang"
---
Docker和Kubernetes都包含了极其庞大的内容，涉及很多领域，很难用一两篇文章将其解释清楚。这篇文章的目的是建立起容器的基本概念和使用场景，了解它与虚拟机的不同之处，并且认识到Kubernetes作为容器集群管理工具，又涉及了哪些功能。

对于一些高级主题，在文章最后附上文档地址，有兴趣可以做延伸阅读。

# References
https://docs.docker.com/engine/understanding-docker/
https://docs.docker.com/engine/userguide/intro/

# Docker
Docker是一种虚拟化技术，这是毋庸置疑的。然而与传统的KVM/Xen/Hyper-V/Vmware等虚拟化技术相比，Docker采用的是非常另辟蹊径的做法。

下面是一张很常见的对比图：
![Docker vs VM](http://gordonsun-blog.s3.amazonaws.com/wp-content/uploads/2015/05/docker-containers-vs-vms.png)

可以看到Docker一直标榜的高效（硬件利用率更高、启动更快）都来自于它取消了Hypervisor，取消了GuestOS这个概念。这张图的信息量其实非常大，做技术的人会想了解Docker Engine是如何做到运行环境隔离的，做产品的人会想知道它怎么就敢去取消Hypervisor这个虚拟化里没有人想过要避开的东西。

IaaS的出现也是为了部署效率和硬件利用率，才选择了虚拟化。不需要去机房布线，不需要去机房开关机。但不要忘了最终虚拟化也是为了应用而服务的，而Docker始终关注着的是应用，IaaS距离应用部署仍然有一段距离。在docker之前，就有人做了Vagrant这个工具，利用IaaS之上再次封装，来提供服务粒度的虚拟化部署镜像封装，虽然底层实现与docker千差万别，但理念却是与docker最为接近。

如果只站在应用的角度，来看下应用所关心的点是什么：
1. 物理资源隔离，端口、内存、cpu等，我要用的就是我的。
2. 运行环境隔离，别的应用需要安装什么版本的库、跑什么进程，跟我不影响。

Docker通过一系列技术手段解决了上面的隔离问题，也就是容器技术。并且带了一系列的好处（以下是拷贝，自行理解）：

**更高效的利用系统资源**

由于容器不需要进行硬件虚拟以及运行完整操作系统等额外开销，Docker 对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更高效。因此，相比虚拟机技术，一个相同配置的主机，往往可以运行更多数量的应用。

**更快速的启动时间**

传统的虚拟机技术启动应用服务往往需要数分钟，而 Docker 容器应用，由于直接运行于宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间。大大的节约了开发、测试、部署的时间。

**一致的运行环境**

开发过程中一个常见的问题是环境一致性问题。由于开发环境、测试环境、生产环境不一致，导致有些 bug 并未在开发过程中被发现。而 Docker 的镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性，从而不会再出现 “这段代码在我机器上没问题啊” 这类问题。

**持续交付和部署**

对开发和运维（DevOps）人员来说，最希望的就是一次创建或配置，可以在任意地方正常运行。

使用 Docker 可以通过定制应用镜像来实现持续集成、持续交付、部署。开发人员可以通过 Dockerfile 来进行镜像构建，并结合 持续集成(Continuous Integration) 系统进行集成测试，而运维人员则可以直接在生产环境中快速部署该镜像，甚至结合 持续部署(Continuous Delivery/Deployment) 系统进行自动部署。

而且使用 Dockerfile 使镜像构建透明化，不仅仅开发团队可以理解应用运行环境，也方便运维团队理解应用运行所需条件，帮助更好的生产环境中部署该镜像。

**更轻松的迁移**

由于 Docker 确保了执行环境的一致性，使得应用的迁移更加容易。Docker 可以在很多平台上运行，无论是物理机、虚拟机、公有云、私有云，甚至是笔记本，其运行结果是一致的。因此用户可以很轻易的将在一个平台上运行的应用，迁移到另一个平台上，而不用担心运行环境的变化导致应用无法正常运行的情况。

**更轻松的维护和扩展**

Docker 使用的分层存储以及镜像的技术，使得应用重复部分的复用更为容易，也使得应用的维护更新更加简单，基于基础镜像进一步扩展镜像也变得非常简单。此外，Docker 团队同各个开源项目团队一起维护了一大批高质量的官方镜像，既可以直接在生产环境使用，又可以作为基础进一步定制，大大的降低了应用服务的镜像制作成本。

当然，有这么多好处，容器技术比起传统Hypervisor的虚拟化技术来说在资源的严格隔离性和安全性上都有所下降。

# 概念

### Docker Engine
![Docker Engine](https://docs.docker.com/engine/article-img/engine-components-flow.png)
Docker Engine provides the core Docker technology that enables images and containers

### Docker 镜像

Docker镜像是用来启动Docker容器的单元，是构建应用的产物。Docker镜像一方面是一个只读的文件系统（通过多个layer构成），另一方面提供了如何启动Docker容器的参数和步骤（通过Dockerfile中的ENV和CMD提供）。

Docker镜像由layer构成， 每一层layer映射到Dockerfile里面即是一条Build命令。

比如使用下面的Dockerfile构建一个新的Docker镜像

```
FROM xxx:2.2.RC2
RUN touch /opt/layertest
RUN echo "Layertest" > /opt/layertest
```

在当前目录使用 docker build -t layertest/1 ./ 构建后，检查layer情况：

```
  [root@env-vt-kube-minion-1 fsp]# docker history layertest/1
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  824a9a414e46        17 seconds ago      /bin/sh -c echo "Layertest" > /opt/layertest    10 B                
  7c558e0a878c        21 seconds ago      /bin/sh -c touch /opt/layertest                 0 B                 
  573757751331        2 weeks ago         /bin/sh -c #(nop) ENTRYPOINT ["/usr/sbin/sshd   0 B                 
  <missing>           2 weeks ago         /bin/sh -c #(nop) EXPOSE 22/tcp                 0 B                 
  <missing>           2 weeks ago         /bin/sh -c /usr/bin/ssh-keygen -A               5.898 kB            
  <missing>           2 weeks ago         /bin/sh -c dos2unix /sbin/service /etc/init.d   5.681 kB            
  <missing>           2 weeks ago         /bin/sh -c chmod 550 /etc/init.d/cron /usr/bi   24.55 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:d1b9f25f11dbb7d78   14.88 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:ba01fe94b0a386124   9.67 MB             
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:723e766dda95d6429   3.04 kB             
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:c383f9f5c8dd25cd9   2.641 kB            
  <missing>           2 weeks ago         /bin/sh -c cd /tmp/all_dependencies_rpms && y   67.13 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) ADD file:2f2e39afe4df9f8edc   20.31 MB            
  <missing>           2 weeks ago                                                         185.3 MB            Imported from -
  [root@env-vt-kube-minion-1 fsp]# docker history xxx:2.2.RC2
  IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
  573757751331        2 weeks ago         /bin/sh -c #(nop) ENTRYPOINT ["/usr/sbin/sshd   0 B                 
  <missing>           2 weeks ago         /bin/sh -c #(nop) EXPOSE 22/tcp                 0 B                 
  <missing>           2 weeks ago         /bin/sh -c /usr/bin/ssh-keygen -A               5.898 kB            
  <missing>           2 weeks ago         /bin/sh -c dos2unix /sbin/service /etc/init.d   5.681 kB            
  <missing>           2 weeks ago         /bin/sh -c chmod 550 /etc/init.d/cron /usr/bi   24.55 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:d1b9f25f11dbb7d78   14.88 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:ba01fe94b0a386124   9.67 MB             
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:723e766dda95d6429   3.04 kB             
  <missing>           2 weeks ago         /bin/sh -c #(nop) COPY file:c383f9f5c8dd25cd9   2.641 kB            
  <missing>           2 weeks ago         /bin/sh -c cd /tmp/all_dependencies_rpms && y   67.13 MB            
  <missing>           2 weeks ago         /bin/sh -c #(nop) ADD file:2f2e39afe4df9f8edc   20.31 MB            
  <missing>           2 weeks ago                                                         185.3 MB            Imported from -
```

因此Docker镜像也是非常轻量级的，每次更新应用版本后实际上并不会将整个image所有的layer刷新，而只是将build application那一层layer更新。

关于Dockerfile的编写参考：

[Build your own image](https://docs.docker.com/engine/getstarted/step_four/)

[Dockerfile best practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)

每个镜像都有一个基础镜像，那么如何从零构建基础镜像？

[Build your own base image](https://docs.docker.com/engine/userguide/eng-image/baseimages/)

### 容器
Docker容器是某个Docker镜像运行时的实例。当一个容器被新建出来后，就有一个所谓的thin layer被加在了Docker镜像layer的最上端，这个layer也被叫做container layer。该容器的读写都发生在这个layer上面，不会影响镜像本身的layer内容。当容器被删除时，这个container layer也一起被删除掉了。每个容器都有自己的container layer，所以基于一个镜像运行多个容器并不会复制多份镜像，而是大家共享同一个镜像，同时由于container layer的存在，每个容器自身又可以保持自己状态。

![](https://docs.docker.com/engine/userguide/storagedriver/images/container-layers.jpg)

关于容器和镜像的更多细节可以参考 [images and containers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)



### Docker Registry
Docker Registry就是一个容器镜像库。做好的容器镜像可以上传至Docker Registry进行发布，用户也可以下载Docker Registry上的镜像至本地运行。

### Docker client
用户操作Docker和与Docker Daemon沟通的主要入口。

### 架构
![Docker Architecture](https://docs.docker.com/engine/article-img/architecture.svg)

使用Docker的主要阶段可以分为：
1. 应用封装：打包自己的应用，将其转换为容器镜像。
2. 镜像发放：将镜像上传至镜像存储节点，如Docker Hub或私有Registry。
3. 部署/运行：在服务器上，下载镜像并运行。

![Docker Engine](https://docs.docker.com/engine/getstarted/tutimg/container_explainer.png)

当你运行上图命令的时候，Docker Engine做了以下几件事：
1. 检查本地是否有hello-world这个docker镜像。
2. 从配置的Registry中下载此镜像到本地。
3. 从镜像创建容器，并运行。


test
