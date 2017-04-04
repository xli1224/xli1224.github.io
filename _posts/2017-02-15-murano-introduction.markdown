# Murano简介
---
layout: post
title: Murano简介
---
Murano的设计目的是为了能够在IaaS平台上获得更好的使用体验。在传统的IaaS平台上，且不提应用的双活和扩容等特性，单是应用的安装体验就非常繁琐，可能连一些VPS厂商都不如。以搭建一个常见的web网站为例，在OpenStack这类IaaS平台上，用户需要先申请一个虚拟机，然后ssh登录，再通过配置软件源或下载包的方式进行所必须的软件安装、配置、调试，然后才能使用应用。

Murano的目的就是将上面所述的步骤自动化，对于用户来说实现一键部署的效果。这也是为何Murano将自身称为Application Catalog的缘故。

为了实现步骤的自动化，Murano要解决两个问题：
1. 云内资源的创建。比如虚拟机、IP、数据卷。
2. 应用的安装、配置。

在OpenStack中，Murano通过对接heat，来实现了资源层面的编排管理。应用的安装配置则通过推送自定义的安装脚本来实现。

Murano中存在两种使用角色：
1. 用户：Murano的使用者，需要创建应用。
2. 开发者：开发应用的Murano安装包

Murano后端架构分为三部分，Murano API，Murano Engine和Murano Agent。前段有murano-dashboard嵌入horizion，命令行也提供了muranoclient。如下图所示：

![Murano Architecture](/static/img/murano-intro/architecture.png)

- Murano API：负责处理用户发来的Rest请求，维护相关的数据结构和Murano自身的业务操作逻辑，如创建环境，添加组件，删除环境。当环境要进行部署时，分解部署任务并发送给Murano Engine。
- Murano Engine：监听并接收Murano API传来的任务，将其下发给Murano Agent执行，将返回值更新至数据库。虽然看起来好像Engine的事情可以由API直接做，但实际上由于MuranoPL的存在，Engine做了DSL和对接heat的工作。
- Murano Agent：监听RaabitMQ，接收任务消息和脚本，执行后返回结果。

# Murano术语
## Environment 环境

官方的解释是：
> An environment is a set of logically connected applications that are grouped together for an easy management. By default, each environment has a single network for all its applications, and the deployment of the environment is defined in a single heat stack. Applications in different environments are always independent from one another.
An environment is a single unit of deployment.

从业务上来说，Environment对应到部署活动时的环境，如Prod，UAT，用户应该为划分好自己的环境。从技术上来说，Environment是对部署组件的隔离，不同环境之间的组件实例（MuranoPL里面的object，不是指真正运行起来的应用）是无法在Murano这一层互相访问的。从逻辑来说，Environment就是一组Application的逻辑集合。

需要特别注意的是，每次部署的最小单位为一个环境。也意味着部署状态的对象针对的是环境，而非某次更改的应用。

## Session 会话
Session是为了支持多个用户同时更改环境而设计的（不是同时部署）。每个用户在创建应用时，实际上并没有将当前应用信息添加到环境的数据中，而是新建了一个Session，拷贝了当前环境的数据后，将新应用的信息写在了Session内。用户点击部署后，触发的部署内容会以当前的Session为基础生成具体的部署任务。部署结束后，Session进入Deployed状态，无法再次被使用。

## Application Package 应用包
应用包的形式为一个.zip文件。应用包指定了应用的部署步骤，实现了应用的部署。

## Library Package 库包
库包与应用包的形式和结构基本一致，它服务于应用包，应用包可借助库包内的方法实现部署。库包不能单独部署。

## Object Model（UI）
应用包和库包内的代码定义各种类，一旦类实例化，就形成了Object Model，Object Model贯穿了应用的生命周期，是Murano解析引擎的执行入口。但是它不对外体现，在Murano API看来，它只是一条json格式的字符串。

从用户开始填写UI开始，生成了一个指定类的Object Model，部署开始后，Engine调用DSL解析包后，从规定的入口调用Object Model的方法，生成更多的Object Model来进行部署操作。部署完成后，Object Model仍然存在，保留了当前状态，提供Action方法供外部掉用。直至应用删除后，Object Model消失。

## Action
当应用部署完毕后，如果需要提供一些对外的接口来操作应用，如启动/停止服务，开发者可以通过定义Action来实现。Action是应用包内入口类的所有指定类型未Action的无参方法，Murano会将这些方法暴露在UI上形成菜单项，供用户使用。

# Murano部署流程
### 上传应用包/库包
通过Horizon界面上传应用包或库包至Murano，在导入时可以为包添加一些元数据属性，也可以不添加，对后续的部署并无影响。
![Upload Application](/static/img/murano-intro/upload-app.png)
![Upload Application](/static/img/murano-intro/upload-app2.png)

### 创建环境
通过Horzion界面创建一个新的环境。默认网络可设置可留空，无影响。
![Add Environment](/static/img/murano-intro/create-env.png)

### 添加应用
在界面上将一个已上传的应用拖进虚线框内，触发应用添加流程。
![Add Application](/static/img/murano-intro/add-app.png)
填写部署应用所需的信息，最终完成应用添加。
![Add Application](/static/img/murano-intro/add-app2.png)
![Add Application](/static/img/murano-intro/add-app3.png)

在这个阶段中，Murano新建了一个Session，并且将新增的应用信息绑定在这个session之上，也就是对于其他用户来说，看到的环境仍然是空的。Session的存在触发了部署按钮的显现。

应用页面是从应用包的UI.yaml读取并转化为html展现出来的，也就是UI文件在这一步才被读取调用。

### 部署环境
点击“部署这个环境”按钮，开始部署。当前Session状态变更为"Deploying"，禁止添加新的应用。等待环境状态变为Ready后，当前Session结束，可以继续部署新的应用。
![Deploy Environment](/static/img/murano-intro/deploy-env.png)

添加应用过程中构建了应用包内的对象实例，部署环境会调用对象的deploy方法，执行应用包内的指令，过程中如果应用包内的指令没有异常抛出，则认为部署成功。

# References
[Developer guide](https://docs.openstack.org/developer/murano/)
[Glossary](https://docs.openstack.org/developer/murano/appendix/articles/specification/murano-api.html)
