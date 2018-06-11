---
title: springboot-datasource
date: 2018-06-11 09:52:09
tags: springboot
catagories: java
---

# springboot配置多数据源，实现Dao层数据源动态切换,并用Druid替换原来的数据源

## ## 1.application.yml中定义两个数据源，这里是测试环境，我们用两个账号模拟上产的主库和只读库。

```yaml
spring:
#读写库
  rw-datasource:
    username: root
    driver-class-name: com.mysql.jdbc.Driver
    password: 123456
    url: jdbc:mysql://192.168.5.44/sell?characterEncoding=utf-8
#只读库
  ro-datasource:
    username: faderwang
    driver-class-name: com.mysql.jdbc.Driver
    password: 123456
    url: jdbc:mysql://192.168.5.44/sell?characterEncoding=utf-8
#    type: com.alibaba.druid.pool.DruidDataSource

  jpa:
    show-sql: true
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
```

## 2.将配置文件中的配置注入到bean中，@Primary注解让spring默认选择masterDataSource这个数据源

```java
@Configuration
@Slf4j
public class DataSourceConfig {


    @Primary
    @Bean("masterDataSource")
    @ConfigurationProperties(prefix = "spring.rw-datasource")
    public DataSource masterDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        return dataSource;
    }

    @Bean("slaveDataSource")
    @ConfigurationProperties(prefix = "spring.ro-datasource")
    public DataSource slaveDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        return dataSource;
    }
}
```

## 3.要实现动态切换数据源，我们还需要借助AbstractRoutingDataSource这个抽象类，我们编写一个实现类，实现它的determineCurrentLookupKey方法，根据对应的lookUpKey获取对应的数据源。

### 3.1创建一个RoutingDataSource继承AbstractRoutingDataSource

```java
public class RoutingDataSource extends AbstractRoutingDataSource{
    @Override
    protected Object determineCurrentLookupKey() {
        return ContextDataBaseHolder.getDataBaseHolder();
    }
}
```

### 3.2这里我们在创建一个类用来存储当前线程的数据源，使用ThreadLocal保证线程安全。这样就可以在Dao层改变ThreadLocal的值从而实现DataSource的动态切换。

> 因为上面我们就是从这个类的？ThredLocal对象中获取的DataSource

```
@Override
    protected Object determineCurrentLookupKey() {
        return ContextDataBaseHolder.getDataBaseHolder();
    }
```



```java
public class ContextDataBaseHolder {

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void setContextHolder(String database) {
        contextHolder.set(database);
    }

    public static String getDataBaseHolder() {
        return contextHolder.get();
    }

    public static void clearDataBaseHolder() {
        contextHolder.remove();
    }
}
```



```java
//这里我们添加一个RoutingDataSource的bean
//告诉spring上线文，使用默认的数据源
    @Bean
    public DataSource primaryDataSource() {
        log.info("create routing datasource...");
        Map<Object, Object> map = Maps.newHashMap();
        map.put("masterDataSource", masterDataSource());
        map.put("slaveDataSource", slaveDataSource());
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(map);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());

        return routingDataSource;
    }

//关键配置
    @Bean
    public PlatformTransactionManager transactionManager() {
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setDataSource(primaryDataSource());
        return transactionManager;
    }
```

