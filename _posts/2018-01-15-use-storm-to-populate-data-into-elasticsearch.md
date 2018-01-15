---
layout: post
title: 使用 Storm 输出至 ElasticSearch
tags:
- Storm
---

其实最大的坑就是storm官方提供的Bolt已经没法用了，就是这玩意儿<https://github.com/apache/storm/tree/master/external/storm-elasticsearch>。里面的RestClient已经和最新的ES6不兼容了，所以不用去折腾这个库了。

Storm的不用，就用ElasticSearch提供的好了。 <https://www.elastic.co/guide/en/elasticsearch/hadoop/current/storm.html>。

如何输出到ElasticSearch里面写的已经比较清楚了。关键在于几个关键配置



## 配置项



### es.storm.bolt.write.ack

写到es以后，ack给上游。默认是false，esbolt会在接收了data之后就返回ack，置为true后，esblot需要确定写入es以后才返回，esbolt需要一直保存数据，因此会增加内存使用量。


### es.storm.bolt.flush.entries.size

每次一个document不会马上写入index，而是存在本地的一个bulk里面，等到一定数量后批量写入。这里就是配置批量写入的触发数值，当设置为1时，等于每个Tuple都被实时写入es。


### es.storm.bolt.tick.tuple.flush

接收一个特定的tick bolt后进行flush，可以从外部触发flush。



### es.mapping.id

这个比较有意思，它涉及到一个问题，就是EsBolt是如何处理Tuple到Document的。就正常情况来说，作为Bolt，收到的Tuple结构会定义在Field声明里，EsBolt通过配置这个id的field名称，会将Tuple中匹配的field作为ElasticSearch里面的id。举例来说，一个Tuple定义了两个字段，id和name，并使用es.mapping.id=id来指定id的mapping，当一个tuple的值为{"id": 1, "name": "test"}时，输出到ElasticSearch里面就会是一个id为1的文档，文档字段为{"id":1, "name":"test"}。如果不指定映射，则文档的值不变，但是id会是ElasticSearch随机生成的值。


### es.input.json

继续上面的问题，从上游发布到ESBolt的Tuple需要指定好文档的结构，但是各种类型的数据都可能会有，每种类型都写一个ESBolt的上游会很难看，过于hard code。

很多场景下面，比如日志分析系统，数据最原始的形态是string，也有很大的可能是json字符串，作为分拣的上游bolt，是无法根据每个字符串里面json字段不同，而动态的声明field，所以只能整个输出到下游。也就是说输出的Tuple字段是类似{"doc": "{\\"id\\":1, \\"name\\": \\"test\\"}"}，它只有一个字段，但是字段的key是无关紧要的，重要的是值是一个json，我们想保存到ElasticSearch里面的是值本事，而不是key加上值。

默认情况下，ESBolt当然会把这个所谓的"doc":"xxx"当成文档的属性存起来，文档看起来会是这个样子：

```java
{
  "doc": {
    "id": 1,
     "name" : "test"
  }
}
```

想要访问id的话，就是doc.id。等于被多嵌套了一层。使用es.input.json=true，ESBolt就会把field的value作为输入，而忽略key，从而减少这层嵌套。当然上游的数据源也必须只输出一个field。


### es.nodes

ElasticSearch服务的地址。


### es.port

ElasticSearch服务的端口。


### es.index.auto.create

当目标index不存在时，创建index。


## 动态输出数据到不同index

当数据内部包含了自己要存到哪个index时，需要解决的问题就是如何输出的问题，有两个途径，一种稍严格但是更强大，另一种简单灵活一些。


### 使用streamId指定ESBolt

整体思路是，预先定义好所有的index，每个index建立一个ESBolt并声明上游的streamid为index。

数据经过一个RoutingBolt的时候，该Bolt解析数据的目标Index，将数据放置在以index为命名的steram里发送到下游，下游对应的ESBlot则会负责将其输出到对应的index里面。

这种做法利用了一个隐含的约定，即每个ESBolt输出的index与它所接收的数据stream名称之间有一对一的关联，RoutingBolt则利用该关联，解析出数据的目标：index后，将其发送向正确的stream。

这种写法最主要的繁琐在于需要事先确定所有的index和type。但是好处是数据已经根据不同的streamId分流了，针对不同index数据的分析和统计可以很顺利的接进来，不需要自己再分辨。

```java
public enum INDEX {
    SHOP_REGISTRY("shop", "registry"),
    SHOP_BINDCARD("shop", "bindcard"),
    SHOP_CANCEL("shop", "cancel"),
    SHOP_ORDER("shop", "order"),
    SHOP_WITHDRA("shop", "withdraw");

    private String app;
    private String business;
    private String index;
    private String type;

    INDEX(String app, String business) {
        this.app = app;
        this.business = business;
        this.index = app + "-" + business;
        this.type = business;
    }

    public String getApp() {
        return app;
    }

    public void setApp(String app) {
        this.app = app;
    }

    public String getBusiness() {
        return business;
    }

    public void setBusiness(String business) {
        this.business = business;
    }

    public String getIndex() {
        return index;
    }

    public void setIndex(String index) {
        this.index = index;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
}
```

```java
public class FilterBolt implements IRichBolt{

    private OutputCollector collector;

    public void execute(Tuple input){
        String log = input.getString(0);
        System.out.println("LOG IS: " + filebeatLog);

        try {
            Gson gson = new Gson();
            Type mapType = new TypeToken<HashMap<String, Object>>() {}.getType();

            HashMap<String, Object> message = gson.fromJson(log, mapType);

            String app = message.get("app").toString().toLowerCase();
            String business = message.get("business").toString().toLowerCase();
            LinkedTreeMap source = (LinkedTreeMap)message.get("source");
            String streamId = app+"-"+business;
            collector.emit(streamId, new Values(gson.toJson(source)));
        } catch (Exception e) { e.printStackTrace();}
        collector.ack(input);
    }

    public void prepare(Map StormConf, TopologyContext context, OutputCollector collector) {
        this.collector = collector;
    }

    public void cleanup() {}

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("source"));
        for (INDEX index : INDEX.values()) {
            declarer.declareStream(index.getIndex().toLowerCase(), new Fields("source"));
        }
    }

    public Map<String, Object> getComponentConfiguration() {return null;}
}
```

```java
public class ESTopology {

    public static void main(String[] args) {
        BrokerHosts hosts = new ZkHosts("127.0.0.1:2181");
        SpoutConfig config = new SpoutConfig(hosts, "es", "/es", UUID.randomUUID().toString());
        config.maxOffsetBehind = 1L;
        config.scheme = new SchemeAsMultiScheme(new StringScheme());
        KafkaSpout kafkaSpout = new KafkaSpout(config);

        TopologyBuilder builder = new TopologyBuilder();
        builder.setSpout("kafka-reader", kafkaSpout);
        builder.setBolt("filter", new FilterBoltBasedType()).shuffleGrouping("kafka-reader");

        INDEX[] indexs = INDEX.values();

        Map esConf = new HashMap();
        esConf.put("es.input.json", "true");
        esConf.put("es.mapping.id", "id");

        for (INDEX index : indexs) {
            EsBolt esBolt = new EsBolt(index.getIndex() + "/" +index.getType(), esConf);
            builder.setBolt("indexer-"+index.getIndex(), esBolt,10).shuffleGrouping("filter", index.getIndex().toLowerCase() );
        }

        Config conf = new Config();
        conf.setDebug(true);
        conf.put("es.nodes", "127.0.0.1");
        conf.put("es.port", 9200);
        conf.put("es.index.auto.create", "true");
        conf.put("es.storm.bolt.write.ack", "true");
        conf.put("es.storm.bolt.flush.entries.size", 1);

        LocalCluster cluster = new LocalCluster();
        cluster.submitTopology("ESIndexer", conf,builder.createTopology());
    }
}
```


### 使用ESBolt的index字段功能

ESBolt还提供了一种方式，可以允许一个ESBlot实例动态的向不同的index输出数据。需要在创建ESBolt的时候，使用index的field代替原本的值，ESBolt就会根据传进来的Tuple上该field的具体值来指定输出的ElasticSearch Index。

在这种情况下，上面例子里的RoutingBolt，就只需要将解析出来的index和type值作为新的字段放在tuple里就行了，而不需要区分stream，下游也只有一个ESBolt。

```java
EsBolt esBolt = new EsBolt( "{index}/{type}", esConf);
builder.setBolt("indexer", esBolt,10).shuffleGrouping("filter" );
```

```java
String app = message.get("app").toString().toLowerCase();
String business = message.get("business").toString().toLowerCase();
LinkedTreeMap source = (LinkedTreeMap)message.get("source");
source.put("index", streamId);
source.put("type", business);
collector.emit(new Values(gson.toJson(source)));
```
