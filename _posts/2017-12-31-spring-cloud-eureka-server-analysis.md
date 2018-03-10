---
layout: post
title: Spring Cloud Eureka Server 源码解析
tags:
- Spring Cloud
---


了解Eureka Server的启动，服务注册，心跳维持和节点同步的工作机制。


## Server 启动

首先从 Spring Boot 入口开始，@EnableEurekaServer 注解如下：

```java
@EnableDiscoveryClient
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({EurekaServerMarkerConfiguration.class})
public @interface EnableEurekaServer {
}
```

默认开启了 EnableDiscoveryClient，并且 Import 了另一个要导入的自动配置，EurekaServerMarkerConfiguration。



```java
@Configuration
public class EurekaServerMarkerConfiguration {

  @Bean
  public Marker eurekaServerMarkerBean() {
    return new Marker();
  }

  class Marker {
  }
}
```

这个类从功能来看没有任何用处，从注释来看主要是添加一个 marker bean，用来激活 Eureka Server 的配置，至于怎么激活，还需要后面在看看。

Spring Boot 的自动配置我们已经知道了，它的入口就是添加进来的依赖包里面的 spring.factories 文件指定的 AutoConfiguration。在 spring-cloud-netflix-eureka-server 包里面，同样也有这个文件。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

在 EurekaServerAutoConfiguration 里面可以看到，上面的 Marker 存在的情况下，此配置才会被激活。这可能是为了避免某些情况下引用了这个包从而导致 EurekaServer 被隐性激活的情况吧。

```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
    InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter
```

在这个类里面，会完成对大部分的功能 Bean 的组装。包括了：

-   EurekaController
-   ServerCodecs
-   InstanceRegistry
-   PeerEurekaNodes
-   一个 JAX-RS 应用的实例
-   一个 FilterRegistrationBean，会把上面的 Application 作为一个 Filter，设置最低优先级并过滤 Eureka Context 下面所有的请求。

Eureka Controller 是一个简单的 Controller，负责的是 eureka dashboard 的显示。包括 Node 自身的信息，以及在该节点上保存的注册信息。ServerCodecs 则是提供了对 json 和 xml 的解析器。

InstanceRegistry 是注册服务的关键。它继承了 PeerAwareInstanceRegistryImpl，看一下它的关键的 registry、cancel、renew

PeerEurekaNodes 维护当前 Eureka Server 要同步的 Peer Node，并通过一个定时任务维护 Peer Node 信息。


<a id="orgaebc0b1"></a>

## 服务注册

\#+NAME InstanceRegistry.java

```java
public class InstanceRegistry extends PeerAwareInstanceRegistryImpl
    implements ApplicationContextAware {

  private ApplicationContext ctxt;
  private int defaultOpenForTrafficCount;

  /** 初始化父类，然后配置自身需要的额外参数，两个参数默认值为 1，尚不确定是否会根据开始 replica 模式自动变化。
    **/
  public InstanceRegistry(EurekaServerConfig serverConfig,
      EurekaClientConfig clientConfig, ServerCodecs serverCodecs,
      EurekaClient eurekaClient, int expectedNumberOfRenewsPerMin,
      int defaultOpenForTrafficCount) {
    super(serverConfig, clientConfig, serverCodecs, eurekaClient);

    this.expectedNumberOfRenewsPerMin = expectedNumberOfRenewsPerMin;
    this.defaultOpenForTrafficCount = defaultOpenForTrafficCount;
  }

  @Override
  public void setApplicationContext(ApplicationContext context) throws BeansException {
    this.ctxt = context;
  }

  //处理注册，leaseDuration 代表服务实例的租期，默认 90s，更新间隔 30s。这三个参数还不确定是否均有 EurekaClient 提供。
  @Override
  public void register(InstanceInfo info, int leaseDuration, boolean isReplication) {
    handleRegistration(info, leaseDuration, isReplication);
    super.register(info, leaseDuration, isReplication);
  }

  @Override
  public void register(final InstanceInfo info, final boolean isReplication) {
    handleRegistration(info, resolveInstanceLeaseDuration(info), isReplication);
    super.register(info, isReplication);
  }

  /**
   将该注册封装成为一个事件发布到 Spring Context 里面，所以还需要去看看有哪些 listener 会监听这个事件。
   **/
  private void handleRegistration(InstanceInfo info, int leaseDuration,
      boolean isReplication) {
    log("register " + info.getAppName() + ", vip " + info.getVIPAddress()
        + ", leaseDuration " + leaseDuration + ", isReplication "
        + isReplication);
    publishEvent(new EurekaInstanceRegisteredEvent(this, info, leaseDuration,
        isReplication));
  }

  private void publishEvent(ApplicationEvent applicationEvent) {
    this.ctxt.publishEvent(applicationEvent);
  }

  private int resolveInstanceLeaseDuration(final InstanceInfo info) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
      leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    return leaseDuration;
  }
}
```

\#+NAME PeerAwareInstanceRegistryImpl

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    //调用 AbstractInstanceRegistry 进行自身注册
    super.register(info, leaseDuration, isReplication);
    //将注册信息同步到其他 Peer 节点
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}

private void replicateToPeers(Action action, String appName, String id,
                              InstanceInfo info /* optional */,
                              InstanceStatus newStatus /* optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        //如果是从其他节点发过来的复制信息，则不做任何事情
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }

        // 否则像当前已知的复制节点同步注册信息
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}

```

```java
 1  public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
 2      try {
 3          //当前线程申请读锁
 4          read.lock();
 5          //获取申请应用在注册表里面的已注册信息
 6          Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
 7          //根据是从其他副本传过来的注册信息还是 client 直接请求到自己这里来的，进行计数。
 8          REGISTER.increment(isReplication);
 9          if (gMap == null) {
10              //是第一次遇到这个应用，初始化注册表项，一个 ConcurrentHashMap。
11              final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
12              gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
13              if (gMap == null) {
14                  gMap = gNewMap;
15              }
16          }
17          //获取已注册的租期信息
18          Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
19          // Retain the last dirty timestamp without overwriting it, if there is already a lease
20          if (existingLease != null && (existingLease.getHolder() != null)) {
21              Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
22              Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
23              logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
24              //如果当前已注册的租期比申请的对象要更晚（更新），那么就用已存在的注册项作为注册对象。
25              if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
26                  logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
27                              " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
28                  logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
29                  registrant = existingLease.getHolder();
30              }
31          } else {
32              // The lease does not exist and hence it is a new registration
33              synchronized (lock) {
34                  //同步方法，更新 expectedNumberOfRenewsPerMin 状态，启动时默认值是 1。因为 client 是每 30s 心跳一次，所以每分钟要+2，然后当前的 renewlPercentThreshold 代表了每分钟接收心跳数的下限，默认是*0.85
35                  if (this.expectedNumberOfRenewsPerMin > 0) {
36                      // Since the client wants to cancel it, reduce the threshold
37                      // (1
38                      // for 30 seconds, 2 for a minute)
39                      this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
40                      this.numberOfRenewsPerMinThreshold =
41                          (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
42                  }
43              }
44  
45          }
46          //组装租期对象
47          Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
48          if (existingLease != null) {
49              //更新服务已启动时间
50              lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
51          }
52          //将此实例的租期对象放到服务的注册项里面
53          gMap.put(registrant.getId(), lease);
54          synchronized (recentRegisteredQueue) {
55              //把当前实例的注册事件放到 queue 里面去，方便查询最近的注册事件
56              recentRegisteredQueue.add(new Pair<Long, String>(
57                                                               System.currentTimeMillis(),
58                                                               registrant.getAppName() + "(" + registrant.getId() + ")"));
59          }
60          // This is where the initial state transfer of overridden status happens
61          if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
62              //如果应用注册信息的 OverrideenStatus 不为 unknown
63              logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
64                           + "overrides", registrant.getOverriddenStatus(), registrant.getId());
65              if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
66                  //并且当前 Map 内不存在该实例的 OverriddenStatus，那么将注册信息里面的状态写入 map
67                  logger.info("Not found overridden id {} and hence adding it", registrant.getId());
68                  overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
69              }
70          }
71          InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
72          if (overriddenStatusFromMap != null) {
73              //如果 map 内存在该实例的 OverrideStatus，那么将该状态放入注册信息里面。
74              logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
75              registrant.setOverriddenStatus(overriddenStatusFromMap);
76          }
77  
78          // Set the status based on the overridden status rules
79          InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
80          registrant.setStatusWithoutDirty(overriddenInstanceStatus);
81  
82          // If the lease is registered with UP status, set lease service up timestamp
83          if (InstanceStatus.UP.equals(registrant.getStatus())) {
84              lease.serviceUp();
85          }
86          registrant.setActionType(ActionType.ADDED);
87          recentlyChangedQueue.add(new RecentlyChangedItem(lease));
88          registrant.setLastUpdatedTimestamp();
89  
90          //失效该实例所代表的 App 的 cache，下次请求返回新内容
91          invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
92          logger.info("Registered instance {}/{} with status {} (replication={})",
93                      registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
94      } finally {
95          read.unlock();
96      }
97  }
```

总结一下，服务注册的时候，Eureka Server 会向 Spring Context 发送一个注册事件，（如果有人感兴趣的话），然后将服务信息更新至自身维护的一个 ConcurrentHashMap，最后看这个注册动作是自己接收到的，还是别的接口同步过来的（isReplication），如果是自己接收到的，还会将这个注册信息同步到其他 node。其他 node 也是 registry 这个方法来处理，只不过对它们来说，这次就是复制注册，而非初始注册。


<a id="org5d125a0"></a>

## 服务心跳维持

服务心跳维持走的是 InstanceRegistry 的 renew 方法。

\#+NAME InstanceRegistry.java

```java
public boolean renew(final String appName, final String serverId,
                     boolean isReplication) {
    log("renew " + appName + " serverId " + serverId + ", isReplication {}"
        + isReplication);

    List<Application> applications = getSortedApplications();
    //从 registry 里面找到与当前对象 id 一直的实例注册信息，并发布一个更新事件。
    for (Application input : applications) {
        if (input.getName().equals(appName)) {
            InstanceInfo instance = null;
            for (InstanceInfo info : input.getInstances()) {
                if (info.getId().equals(serverId)) {
                    instance = info;
                    break;
                }
            }
            publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
                                                        instance, isReplication));
            break;
        }
    }
    //调用上层的更新
    return super.renew(appName, serverId, isReplication);
}
```

\#+NAME PeerAwareInstanceRegistry.java

```java
public boolean renew(final String appName, final String id, final boolean isReplication) {
    if (super.renew(appName, id, isReplication)) {
        //如果自身更新成功，再把这个心跳同步到其他 Peer 节点
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```

\#+NAME AbstractInstanceRegistry.java

```java
public boolean renew(String appName, String id, boolean isReplication) {
    RENEW.increment(isReplication);
    //从 registry 里面取出对应服务的注册信息（包括所有实例）
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    //获取该实例的租期信息
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {
        InstanceInfo instanceInfo = leaseToRenew.getHolder();
        if (instanceInfo != null) {
            // touchASGCache(instanceInfo.getASGName());
            //根据当前状态判断要覆盖的状态
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                                                                                       instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                            + "; re-register required", instanceInfo.getId());
                RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }
            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                Object[] args = {
                    instanceInfo.getStatus().name(),
                    instanceInfo.getOverriddenStatus().name(),
                    instanceInfo.getId()
                };
                logger.info(
                            "The instance status {} is different from overridden instance status {} for instance {}. "
                            + "Hence setting the status to overridden status", args);
                instanceInfo.setStatus(overriddenInstanceStatus);
            }
        }
        //更新计数器
        renewsLastMin.increment();
        //更新租期的时间戳
        leaseToRenew.renew();
        return true;
    }
}
```

同步的逻辑基本与注册一致，根据心跳来源更新对应实例的租期，并且将心跳同步至其他 Peer 节点。


<a id="orgdf27365"></a>

## Peer Node 添加

在初始化的时候，Context 构建阶段会调用 PeerEurekaNodes 的 start 方法，在这里会初始化与其他 Peer 节点的连接。

```java
public void start() {
     taskExecutor = Executors.newSingleThreadScheduledExecutor(
             new ThreadFactory() {
                 @Override
                 public Thread newThread(Runnable r) {
                     Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                     thread.setDaemon(true);
                     return thread;
                 }
             }
     );
     try {
         updatePeerEurekaNodes(resolvePeerUrls());
         Runnable peersUpdateTask = new Runnable() {
             @Override
             public void run() {
                 try {
                     updatePeerEurekaNodes(resolvePeerUrls());
                 } catch (Throwable e) {
                     logger.error("Cannot update the replica Nodes", e);
                 }

             }
         };
         //创建一个单线程定时任务，定期更新 Peer 节点信息
         taskExecutor.scheduleWithFixedDelay(
                 peersUpdateTask,
                 serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                 serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                 TimeUnit.MILLISECONDS
         );
     } catch (Exception e) {
         throw new IllegalStateException(e);
     }
     for (PeerEurekaNode node : peerEurekaNodes) {
         logger.info("Replica node URL:  " + node.getServiceUrl());
     }
 }
```

updatePeerEurekaNodes 会根据 client 配置项里面的 URL 信息，对比当前维护的 PeerNodes，如果有新增或删除，则生成新的 PeerNodes，否则不变。
