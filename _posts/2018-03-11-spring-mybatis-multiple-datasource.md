<<<<<<< HEAD
=======

>>>>>>> f9a55b93d7ccab99f212b04dbac71c52d0c22c6f
---
layout: post
title: Mybatis 在 Spring 下使用多数据源
tags:
- Spring
- Java
---

从去年10月开始项目较忙，感觉大部分时间都是在做一些小的 trick，补一些遗漏的知识点或者说工具点而已。很少能有机会做一些有意思的钻进去的东西，其实在备选 topic 里面已经储备了好多要看的东西，也许4月份开始能有时间腾出来解决这些问题。



今天的主题也很简单，就是一个 Spring Application 连接多个数据源的问题。一部分 Mapper 使用数据源 A，另一部分 Mapper 使用数据源 B。就是在 MapperScan 的时候，指定不同的 SqlSessionFactory即可。

## 使用 Spring

```xml
<bean id="sqlSessionFactoryMaster" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:conf/mybatis.xml"></property>
        <property name="dataSource" ref="dataSourceMaster"/>
        <property name="mapperLocations" value="classpath*:cn/**/dao/master/*.xml"/>
        <property name="typeAliasesPackage" value="${aliases.modelpackages}"/>
    </bean>


    <bean id="orderScannerConfigurerMaster" class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="cn/**/dao/master"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactoryMaster"/>
        <property name="properties">
            <value>
                mappers=tk.mybatis.mapper.common.Mapper
            </value>
        </property>
    </bean>

<bean id="sqlSessionFactorySlave" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:conf/mybatis.xml"></property>
        <property name="dataSource" ref="dataSourceSlave"/>
        <property name="mapperLocations" value="classpath*:cn/**/dao/slave/*.xml"/>
        <property name="typeAliasesPackage" value="${aliases.modelpackages}"/>
    </bean>


    <bean id="orderScannerConfigurerSlave" class="tk.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="cn/**/dao/slave"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactorySlave"/>
        <property name="properties">
            <value>
                mappers=tk.mybatis.mapper.common.Mapper
            </value>
        </property>
    </bean>
```



## 使用 Spring Boot

主要是需要创建两个Configuration，里面分别实现了 DataSource 的创建（读取application yml 里面特定的属性），TransactionManager 的创建，以及最重要的，sqlSessionFactory 的创建。通过创建不同的 sqlSessionFactory，配置读取不同的 mapper 文件。

```java
@Configuration
@MapperScan(basePackages = MasterDatasourceConfig.Package, sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterDatasourceConfig {

    static final String Package = " cn.*.master";

    static final String MAPPER_LOCATION = "classpath:mapper/master/*.xml";

    @Value("${master.datasource.url}")
    private String url;

    @Value("${master.datasource.username}")
    private String user;

    @Value("${master.datasource.password}")
    private String password;

    @Value("${master.datasource.driver}")
    private String driverClass;

    @Bean(name = "masterDataSource")
    @Primary
    public DataSource masterDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(this.driverClass);
        dataSource.setUrl(this.url);
        dataSource.setUsername(this.user);
        dataSource.setPassword(this.password);
        return dataSource;
    }

    @Bean(name = "masterTransactionManager")
    @Primary
    public DataSourceTransactionManager masterTransactionManager(){
        return new DataSourceTransactionManager((masterDataSource()));
    }

    @Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDataSource") DataSource masterDataSource)
            throws Exception {
        final SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(masterDataSource);
        sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
        .getResources(MasterDatasourceConfig.MAPPER_LOCATION));

        return sessionFactoryBean.getObject();
    }
}
```

```java
@Configuration
@MapperScan(basePackages = SlaveDatasourceConfig.Package, sqlSessionFactoryRef = "slaveSqlSessionFactory")
public class SlaveDatasourceConfig {

    static final String Package = "cn.*.slave";

    static final String MAPPER_LOCATION = "classpath:mapper/slave/*.xml";

    @Value("${slave.datasource.url}")
    private String url;

    @Value("${slave.datasource.username}")
    private String user;

    @Value("${slave.datasource.password}")
    private String password;

    @Value("${slave.datasource.driver}")
    private String driverClass;

    @Bean(name = "slaveDataSource")
    public DataSource slaveDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(this.driverClass);
        dataSource.setUrl(this.url);
        dataSource.setUsername(this.user);
        dataSource.setPassword(this.password);
        return dataSource;
    }

    @Bean(name = "slaveTransactionManager")
    public DataSourceTransactionManager slaveTransactionManager(){
        return new DataSourceTransactionManager((slaveDataSource()));
    }

    @Bean(name = "slaveSqlSessionFactory")
    public SqlSessionFactory slaveSqlSessionFactory(@Qualifier("slaveDataSource") DataSource slaveDataSource)
            throws Exception {
        final SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(slaveDataSource);
        sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
        .getResources(SlaveDatasourceConfig.MAPPER_LOCATION));

        return sessionFactoryBean.getObject();
    }
}
```

如果说是单数据源的情况下，绑定 mybatis 是非常简单的。配置文件里面配置 spring datasource即可，其他都是自动的，包括创建 SqlSessionFactory。