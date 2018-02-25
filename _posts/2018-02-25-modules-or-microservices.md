- [微服务解决了什么问题？](#orgcb8ccd4)
  - [Java 的模块有什么特点，和用 jar 有什么区别](#org452b136)
  - [微服务的实施问题。](#org9d43fc6)
  - [微服务的优势](#org4ce224a)
  - [模块的优势](#orgf236762)
  - [对于很多急着上微服务的公司的建议](#org3a60a18)

今天看了 Sander Mak 关于微服务和模块化的模块，里面有些论点非常有趣，总结一下，方便不想看55分钟视频的人。

他的主要论点是，是否每个想摆脱 monolith style 应用的人，都必须要选择微服务？微服务解决了什么问题，又带来了什么问题，模块化能不能算作一个其他选项（不一定是更好）？


<a id="orgcb8ccd4"></a>

# 微服务解决了什么问题？

-   Manage complexity。将复杂的应用拆分，主要能够实现并行开发，降低开发和维护的复杂度。
-   Scaling。能够细粒度的水平扩展。
-   Resilience。可靠性高，每个服务独立部署，单个服务失效不影响其他服务。


<a id="org452b136"></a>

# Java 的模块有什么特点，和用 jar 有什么区别

使用模块和 jar 的区别：

1.  jar 包不能明确指定依赖，只有当某个 class load 时才会发现 class 是否存在。
2.  jar 包是被线性的放在 classpath 里面，没有边界，所有 classpath 上的 jar 包都是可用的，所有 jar 包里面的 public class 都是可以被使用的。

模块有几个优点：

1.  module resolution. 可靠解决依赖问题。
2.  increased security。可以选择只暴露哪些接口，而非整个 jar 包的所有类。

JVM 在编译阶段就可以控制模块边界，避免使用未发布的接口。


<a id="org9d43fc6"></a>

# 微服务的实施问题。

微服务是有很高的使用成本的，服务治理、断路器、配置管理、网关等等配套技术需要学习。module 设计可以有助于程序推迟进入微服务架构的阶段。


<a id="org4ce224a"></a>

# 微服务的优势

-   Choose best stack for each service。也要看公司有没有这个能力去驾驭多种技术栈。
-   Independent deployment。需要控制服务间的版本依赖（想想 openstack 各种功能模块的混合，对于 api 版本的控制），在允许的情况下，可以混合微服务和模块化技术，将耦合的业务放入一个应用，以模块化区分，并行开发，然后统一发布。各服务都是自治的。
-   Independent failure。当服务不是自治的时候，一个服务的失效，同样也会引起其他服务失效。除非用断路器/fallback。
-   Independent scaling。需要好好确认一下，是否真的有一部分功能确实需要独立出来，因为在流量变大时它与其他模块有着显著不同的性能要求。当这个不为真的时候，所有的功能其实都需要差不多的性能或者实例数，将其都放在一个应用里面，在分布式的环境下，其实效果又能差多少呢？假如整个应用分为3个微服务，每个微服务都需要两个实例（即使实例的规格不同），但是如果我们将三个微服务合并为一个应用，同样放在两个实例下面，甚至更多实例下面，整个应用仍然是在 scaling 的。


<a id="orgf236762"></a>

# 模块的优势

-   Ease of deployment and management. 这个不用说了，参考单体应用的部署方式。
-   Strong but refactorable boundaries. 很多时候现有的微服务并不完全合适复用，由于团队之间的沟通比较困难，往往后来者会倾向于开发一个自己的版本的微服务，造成微服务不仅没有得到复用，并且数量越来越多。Moduler 的复用性会更好。
-   compiler knows about your boundaries. Module 的强制性是由 compiler 提供的，而微服务则往往靠文档约定。
-   Strongly typed, In-process communication. No seiralization or network latency. 请求更好处理，微服务往往要考虑同步/异步以及各种序列化和重试策略的问题。
-   Eventual consistency is a choice. 使用模块化设计的时候，仍然建议将数据分区，不同模块有自己的后端数据库。
-   Excplicit Dependencies。 Less Run-time failure modes. 能够在编译阶段识别依赖，尽量避免运行时错误。


<a id="org3a60a18"></a>

# 对于很多急着上微服务的公司的建议

要好好考虑下微服务所解决的问题，是否真的是自己当前已经遇到的问题，以及自己是否真的已经具备了实施微服务的能力。

don't try to solve prolbems that you dont have. solve problems you do have in the simplest possible way.

只要遵守下面三条设计原则去设计你的模块，后面也可以很轻松的将模块转化为微服务。

-   strong encapsulation
-   well-defined interfaces
-   explicit dependencies

当你的应用只是需要拆分边界的时候，提升开发和维护效率时，模块完全可以替代微服务。而当你的模块数量上升，各个模块的业务压力不均匀，集群体量越来越大，业务的上线频度越来越不同的时候，你才需要转向微服务。
