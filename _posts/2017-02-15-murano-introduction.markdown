# Murano 简介

Murano的设计目的是为了能够在IaaS平台上获得更好的使用体验。在传统的IaaS平台上，且不提应用的双活和扩容等特性，单是应用的安装体验就非常繁琐，可能连一些VPS厂商都不如。以搭建一个常见的web网站为例，在OpenStack这类IaaS平台上，用户需要先申请一个虚拟机，然后ssh登录，再通过配置软件源或下载包的方式进行所必须的软件安装，配置，最后使用应用。

Murano的目的就是将上面所述的步骤自动化，对于用户来说实现一键部署的效果。这也是为何Murano将自身称为Application Catalog的缘故。

为了实现步骤的自动化，Murano要解决两个问题：
1. 云内资源的创建。比如虚拟机、公网IP、外部卷
2. 应用的安装、配置。

在OpenStack中，Murano通过对接heat，来实现了资源层面的编排管理。应用的安装配置则通过推送自定义的安装脚本来实现。

Murano的架构分为三部分，Murano API，Murano Engine和Murano Agent。

- Murano API：负责处理用户发来的Rest请求，维护相关的数据结构和Murano自身的业务操作逻辑，如创建环境，添加组件，删除环境。当环境要进行部署时，分解部署任务并发送给Murano Engine。
- Murano Engine：监听并接收Murano API传来的任务，将其下发给Murano Agent执行，将返回值更新至数据库。虽然看起来好像Engine的事情可以由API直接做，但实际上由于MuranoPL的存在，Engine做了DSL和对接heat的工作。
- Murano Agent：监听RaabitMQ，接收任务消息和脚本，执行后返回结果。

# Murano术语
## Environment 环境

官方的解释是：
> An environment is a set of logically connected applications that are grouped together for an easy management. By default, each environment has a single network for all its applications, and the deployment of the environment is defined in a single heat stack. Applications in different environments are always independent from one another.


每一次部署就是以一个环境为单位。
从业务上来说，Environment对应到部署活动时的环境，如Prod，UAT，用户应该为划分好自己的环境。从技术上来说，Environment是对部署组件的隔离，不同环境之间的组件实例（MuranoPL里面的object，不是指真正运行起来的应用）是无法在Murano这一层互相访问的。
# Application Package


# Environment
An environment is a set of logically connected applications that are grouped together for an easy management. By default, each environment has a single network for all its applications, and the deployment of the environment is defined in a single heat stack. Applications in different environments are always independent from one another.

An environment is a single unit of deployment. This means that you deploy not an application but an environment that contains one or multiple applications.



# Object Model（UI）
Murano的动态UI由UI.yaml控制，UI.yaml主要包含4部分：

- Version
- Template
- Application
- Forms

Version指定此UI文件的语言版本，这样解释器才能确定自己是否可以正确解析。

Template和Application一起构成了应用对象模型（Application Object Model），用户在Dashboard上的输入最终都是为了组装为一个对象模型，并以json格式发送给Engine。

https://murano.readthedocs.io/en/stable/murano_pl/murano_pl_index.html

http://docs.openstack.org/developer/murano/draft/appdev-guide/murano_pl.html

http://yaql.readthedocs.io/en/latest/getting_started.html#basic-yaql-query-operations

# Workflow (Engine)

# Template (Agent Unit)
