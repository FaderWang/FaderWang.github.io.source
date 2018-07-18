---
title: SpringBoot MyBatis集成 
date: 2018-07-17 10:55:29
tags: springboot, mybatis
categories: java
---

### SpringBoot快速开发，使用jpa是最方便的，但是如果业务逻辑相对复杂，需要定制化sql的情况下，我们就需要使用mybatis、hibernate等框架。

#### 列出需要用到的所有maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.10</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

#### 首先我们进行mybatis相关配置，正常情况下我们需要在resource目录下新建一个mybatis-config.xml的配置文件，进行mybatis一些策略的配置。

> 下图是一个完整的setting配置 

```xml-dtd
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>
```

在Spring Boot中，我们可以application.properties配置，从而代替mybatis-config.xml配置文件

这里是我配置的几个属性，具体的作用就不做阐释了

```xml
#mybatis配置
mybatis.configuration.use-generated-keys=true
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.use-column-label=true
mybatis.type-aliases-package=com.faderw.school.domain
```

#### 然后配置数据库相关属性，同样也在application.properties文件中配置，*spring.dataource.type*配置数据源的实现类型,这里我们用DruidDataSource代替默认的数据源

```xml
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.url=jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull&useSSL=false
```

#### 创建一个配置类DataSourceConfig,@MapperScan会自动扫描包下面的类，代替@mapper注解，告诉spring这是一个mapper。然后配置一个datasource的bean以及sqlSessionFactory的bean，注意@ConfigurationProperties这个注解将会根据prefix属性自动装配application.porperties里面的配置

```java
@Configuration
@MapperScan("com.faderw.school.dao")
public class DataSourceConfig {

    private static final String SPRING_DATASOURCE = "spring.datasource";
    private static final String MAPPER_LOCATIONS = "mapper/*.xml";

    @Value("${mybatis.type-aliases-package}")
    private String typeAliasesPackage; 

	@Bean(name = "dataSource")
    @ConfigurationProperties(prefix = SPRING_DATASOURCE)
    public DruidDataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        return dataSource;
    }

    @Bean(name = "sqlSessionFactory")
    public SqlSessionFactoryBean sessionFactoryBean() throws IOException{
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource());

        String patternPath = ResourcePatternResolver.CLASSPATH_URL_PREFIX + MAPPER_LOCATIONS;
        sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(patternPath));
        sessionFactoryBean.setTypeAliasesPackage(typeAliasesPackage);

        return sessionFactoryBean;
    }
}
```

这里我们并没有配置事物管理器，因为springboot的自动装配会为我们默认配置DataSourceTransactionManager，我们只需要在程序入口类加上@EnableTransactionManagement开启注解事物的支持。