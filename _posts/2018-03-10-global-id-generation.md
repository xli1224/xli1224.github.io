---
layout: post
title: 全局唯一自增 ID 生成方案
tags:
- Java
---

## 需求

所有业务共用一个序列 ID，要求 ID 必须从0开始，要求 ID 必须自增，要求 ID 能够归零。所以类似 UUID 或者 snowflake 这种全局 ID 生成的方案就不在考虑了，必须依靠数据存储完成。

## 数据库自增 ID

利用数据库的自增键，所有需要申请 ID 的业务，统一通过数据库事务进行锁定并获取值。

```sql
create table counter (
`id` INT UNSIGNED AUTO_INCREMENT,
`stub` char(1),
PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

配置 mybatis 获取自增 id

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="me.xiang.demo.idgenerator.mapper.IDMapper">
    <cache/>

    <sql id="TABLE_NAME">counter</sql>

    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        insert into
        <include refid="TABLE_NAME"/>
        (stub) values (#{stub})
    </insert>
</mapper>
```



用 mybatis-spring-boot-starter 来帮助配置 datsource、sqlSessionFactory 以及 mapper scan。

```properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/local?user=xiang&password=123456
mybatis.mapperLocations=classpath:me.xiang.demo.idgenerator.mapper/*.xml
```



创建自增 ID 服务。

```java
@Component
public class MybatisIDService implements IDService{

    @Autowired
    IDMapper idMapper;

    public long getNextID() {
        ID stub = new ID('a');
        idMapper.insert(stub);
        return stub.getId();
    }
}
```



## redis 自增 ID

使用 Redis 提供的原子 INCR 操作，维护一个计数器，并以计数器的值作为 ID。

使用 spring-data-redis 配置一个 RedisAtomLong 对象，原子性更新并获取值。

```properties
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=127.0.0.1
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
```



由于 RedisAtomLong 要求必须以 RedisTemplate\<String, Long\>作为参数，所以需要手动构建此 bean，spring-data-redis 已经提供了 RedisConnectionFactory。

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Long> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate <String, Long> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<Long>(Long.class));
        return template;
    }
}
```



创建 ID 自增服务

```java
@Component
public class RedisIDService implements IDService{

    RedisAtomicLong atomicLong;

    @Autowired
    public RedisIDService(RedisTemplate<String, Long> idTemplate){
        this.atomicLong = new RedisAtomicLong("counter", idTemplate);
    }

    public long getNextID(){
        return atomicLong.incrementAndGet();
    }
}
```



## 分段式自增 ID

因为无论是 MySQL 还是 Redis，都涉及到竞争的问题。Redis 的原子变量只是把多并发的获取变成了单线程的操作。MySQL 则是数据库对于不同线程进来的事务进行锁竞争（这里也比较复杂，涉及到 InnoDB 对于不同自增模式下，锁的实现不同，但仍然有锁）。如果说 IO 不是关键压力的话，那么还有可能发生在竞争上面。中心锁的一种常见解决方案就是分段发放，即一次发放一批，省去频繁申请的过程，这样带来的结果就是 ID 可能出现空洞，即发了没用完又来申请的情况，要看业务是否可以容忍。

按照这个思路，后端是数据库还是 Redis 都可以。主要是客户端需要维护一个 ID 段进行发放，发放完成后再次进行申请。数据库的话，需要维护一条记录已发放的 ID 即可，多客户端竞争依靠数据库的行级锁。Redis 使用一个锁和一个计数器进行配合。客户端每次更新计数器前申请锁，更新后释放。或者省事一点，spring-data-redis 的 RedisAtomLong 也同样支持原子更新。

客户端维护 ID 段又涉及到了本地的多线程竞争，原子计数器的更新不是问题，问题在于判断是否要申请新的 ID 段。这可能导致分配方法本身要做成一个同步方法，更好的办法就是用 CAS 来解决这个问题，不要依赖于锁的阻塞。



使用 Redis 的客户端 ID 分段方案。

```java
package me.xiang.demo.idgenerator.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.support.atomic.RedisAtomicLong;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicLong;

@Component
public class RedisSegmentIDService implements IDService {

    RedisAtomicLong atomicLong;

    AtomicLong currentId = new AtomicLong(-1L);
    AtomicLong start = new AtomicLong(-1L);
    AtomicLong end = new AtomicLong(-1L);

    @Autowired
    public RedisSegmentIDService(RedisTemplate<String, Long> idTemplate){
        this.atomicLong = new RedisAtomicLong("counter", idTemplate, 0L);
    }

    public long getNextID(){
        if (currentId.get() > end.get()){
            getNextSegment();
        }

        long tempId = currentId.get();
        while (!currentId.compareAndSet(tempId, tempId+1)) {
            tempId = currentId.get();
        }
        return tempId+1;
    }

    private void getNextSegment2(){
        long tempEnd = end.get();
        //同时进入 if 条件可能造成 redis 申请了一批但是由于 CAS 放弃了。可以考虑将方法加锁，毕竟申请操作频度低很多，while 循环还需要浪费判断条件。
        while (currentId.get() > end.get()) {
            if (end.compareAndSet(tempEnd, atomicLong.addAndGet(1000))) {
                start = new AtomicLong(end.get() - 1000);
            }
        }
    }

    private synchronized void getNextSegment(){
        if (currentId.get() > end.get()) {
            end.set(atomicLong.addAndGet(1000));
            start = new AtomicLong(end.get() - 1000);
        }
    }

    public void print(){
        System.out.println("CurrentID is: " + currentId.get());
        System.out.println("start is: " + start.get());
        System.out.println("end is: " + end.get());
    }
}

```





## 性能对比

创建一批线程，线程内部限时1分钟，循环申请ID，看看单位时间内的 ID 发放数量对比。

```java
public class IDTask implements Runnable{

    private IDService idService;

    private int counter;

    private Long start = 0L;

    public IDTask(IDService idService) {
        this.idService = idService;
        this.counter = 0;
    }

    public void run() {
        this.start = System.currentTimeMillis();
        String threadName = Thread.currentThread().getName();
        while (true) {
            try {
                Thread.sleep(1L);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            this.idService.getNextID();
            counter += 1;

            if (timeout()) {
                IDTaskManager.addCounter(threadName, counter);
                return;
            }
        }
    }


    public boolean timeout() {
        return (System.currentTimeMillis() - start) > 60*1000L;
    }
}
```



```java
@Component
public class IDTaskManager {

    private static int counter = 0;

    @Autowired
    private RedisSegmentIDService idService;

    public static synchronized void addCounter(String threaName, int value) {
        counter += value;
        System.out.println("Updated Counter: " + counter);
    }

    public void start(int threadCount) throws InterruptedException{
        counter = 0;
        System.out.println("START THREAD COUNT: " + threadCount);
        ThreadFactory namedThreadFactory = new ThreadFactoryBuilder().setNameFormat("id-task-thread-%d").build();
        ExecutorService executor = Executors.newFixedThreadPool(threadCount, namedThreadFactory);
        int i = 0;
        while (i < threadCount) {
            IDTask idTask = new IDTask(idService);
            executor.submit(idTask);
            i++;
        }
    }
}
```



mysql （Hikari 连接池）：

100 线程： Updated Counter: 150366

200 线程： Updated Counter: 159485

300 线程： Updated Counter: 150012



redis：

100 线程：Updated Counter: 1399911

200 线程：Updated Counter: 1412502

300 线程：Updated Counter: 1459106




redis 分段：

100 线程：Updated Counter: 4822451
200 线程：Updated Counter: 7904208
300 线程：Updated Counter: 6775957(竞争太激烈导致发放降低，一般来说一个 JVM 不会同时300个线程抢占的情况)



## 总结

分段的性能还是明显要好不少的，只要能接受由于程序重启等原因出现 ID 空洞的情况。Redis 的情况相对来说更简洁直观，性能折中。MySQL 适用于不想让应用架构里多出个 Redis，同时对性能要求也不高的情况。