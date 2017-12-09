---
layout: post
title: 使用代码更新 Log4j2 的日志配置
tags:
- Java
---

最近有一个业务需求，需要提取业务日志数据进行运营分析。由于之前各个应用对于业务数据本来打印的就少，所以想借此机会，提供一个接口来实现业务数据的专用日志，与一般的运维日志消息区分开来。

考虑到侵入性的问题，不希望由具体开发组去配置日志框架，而是通过统一的工具包来实现。现在公司的绝大部分产品用的都是的 log4j2 作为实现，就来看看这个怎么配好了。

Log4j2 的架构如下： ![img](/assets/images/Log4jClasses.jpg)

里面几个主要的组件：

-   Logger
-   LoggerConfig
-   Appender

Logger/LoggerConfig/Appender 都是可以用 name 指定的。

当使用 LogManager 获取 Logger 是，要使用一个具体的名字，LogManager 会根据这个名字去寻找合适的 LoggerContext 并且从中获取一个 Logger。如果 Logger 不存在，则创建的时候会根据以下规则寻找到一个 LoggerConfig，并关联到新建的 Logger。

-   与 Logger 的名字相同
-   属于 Logger 的父包
-   根 LoggerConfig

LoggerConfig 关系到将日志事件输出到哪个 Appender。

我们现在需要做的其实就是通过代码，而非是配置文件的方式去新建一个 LoggerConfig 和 Appender，将其交给 LoggerContext 并更新。在官方给出的例子里面大多数是从一开始就 build 一个全新的 Context，只有一个提到了如何去更新 context，这就够了，不一样的细节可以自己去琢磨。

```java
final LoggerContext ctx = (LoggerContext) LogManager.getContext(false);
final Configuration config = ctx.getConfiguration();
Layout layout = PatternLayout.createLayout(PatternLayout.SIMPLE_CONVERSION_PATTERN, config, null,
                                           null,null, null);
Appender appender = FileAppender.createAppender("target/test.log", "false", "false", "File", "true",
                                                "false", "false", "4000", layout, null, "false", null, config);
appender.start();
config.addAppender(appender);
AppenderRef ref = AppenderRef.createAppenderRef("File", null, null);
AppenderRef[] refs = new AppenderRef[] {ref};
LoggerConfig loggerConfig = LoggerConfig.createLogger("false", "info", "org.apache.logging.log4j",
                                                      "true", refs, null, config, null );
loggerConfig.addAppender(appender, null, null);
config.addLogger("org.apache.logging.log4j", loggerConfig);
ctx.updateLoggers();
```

从上面的代码里面可以看到，要动态配置一个已经初始化好的 LoggerContext，基本分为以下几步：

1.  读取当前 Context；
2.  创建新的 Appender 实例（layout 只是 FileAppender 的一个必须参数，其他 Appender 也有更多的类型需要配置）；
3.  创建新的 LoggerConfig；
4.  将 Appender 实例挂载到 LoggerConfig 里面；
5.  将 LoggerConfig 注册进 Context 并更新 Context。

官网的例子只是更新了一个使用普通 FileAppender 的 Logger，在正常环境下，都会选择用 RollingFileAppender 来实现对日志的自动清理。用代码创建 RollingFileAppender 稍微要麻烦一点。

先看 RollingFileAppender 的创建方法：

```java
public static RollingFileAppender createAppender(
        // @formatter:off
        final String fileName,
        final String filePattern,
        final String append,
        final String name,
        final String bufferedIO,
        final String bufferSizeStr,
        final String immediateFlush,
        final TriggeringPolicy policy,
        final RolloverStrategy strategy,
        final Layout<? extends Serializable> layout,
        final Filter filter,
        final String ignore,
        final String advertise,
        final String advertiseUri,
        final Configuration config) 
```

关键的组件是 TriggerPolicy 以及 RolloverStrategy。TriggerPolicy 的实现有 SizeBasedTriggeringPolicy 和 TimeBasedTriggeringPolicy，Policy 负责实现判断是否满足 Rollover 的条件。如果满足的话，FileManager 会触发 rollover 动作，而如何 rollover 恰好是 RolloverStrategy 决定的。

以 DefaultRolloverStrategy 为例：

```java
public static DefaultRolloverStrategy createStrategy(
            // @formatter:off
            @PluginAttribute("max") final String max,
            @PluginAttribute("min") final String min,
            @PluginAttribute("fileIndex") final String fileIndex,
            @PluginAttribute("compressionLevel") final String compressionLevelStr,
            @PluginElement("Actions") final Action[] customActions,
            @PluginAttribute(value = "stopCustomActionsOnError", defaultBoolean = true)
                    final boolean stopCustomActionsOnError,
            @PluginConfiguration final Configuration config)
```

max 和 min 决定了最多和最少会有几个 rollover 文件存在，超出范围的 rollover 文件会被清理掉。但是在使用 TimeBasedTriggeringPolicy 时会发现 rollover 的文件一直存在不会被 max 所限制住。想要自动清理超期的问题，就得借助自定义的 action。

```java
PathCondition lastModified = IfLastModified.createAgeCondition(Duration.parse(totalTimeLimit), null);
PathCondition fileNameMatch = IfFileName.createNameCondition(
                                                             cleanFilePattern,
                                                             null,
                                                             null
                                                             );
PathCondition[] conds = new PathCondition[] {fileNameMatch, lastModified};

DeleteAction action = DeleteAction.createDeleteAction(
                                                      fileDir,
                                                      false,
                                                      1,
                                                      false,
                                                      null,
                                                      conds,
                                                      null,
                                                      config
                                                      );
Action[] actions = new Action[]{action};
```

有了 action，就可以创建出可用的 strategy，剩下的 Appender、LoggerConfig 以及 LoggerContext 的更新都与官方的例子没什么区别了。
