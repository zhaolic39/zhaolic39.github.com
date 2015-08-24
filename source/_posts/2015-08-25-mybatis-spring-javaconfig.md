---
layout: post
title: Spring下的Mybatis配置
date: 2015-08-19 00:05:58
comments: true
categories: java
---
`J2EE`开发界中一个永远避免不了的争论就是`Hibernate`与`Mybatis`哪个好。虽然我自己倾向`Hibernate`，但这两种`ORM`框架我也都没有在比较大型的场景上使用过，没有资格做评价。

最近公司有一个项目，主程让我用`Spring`与`Mybatis`搭建服务端框架，这里就做一下简单的学习记录。
## Java Config方式的Mybatis启动配置
前一篇中提到，公司推荐`Spring`使用`Java Config`的方式进行配置，所以`Mybatis`也不例外。  
先看一段网上找到比较常见的xml方式配置：
```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
</bean>

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <!--指定实体类映射文件，可以指定同时指定某一包以及子包下面的所有配置文件 -->  
    <property name="mapperLocations"  value="classpath*:com/nd/mathlearning/server/*/dao/mapper/*.xml"/>
    <!--mybatis全局配置文件 -->  
    <property name="configLocation" value="classpath:mybatis/mybatis-config.xml" />
    <property name="dataSource" ref="dataSource" />
</bean>
```
配置的核心是`sqlSessionFactory`，将`dataSource`数据源注入就完成了`Mybatis`会话工厂的实例化了。  
接着把这段配置翻译成`Java config`版本：
```java
@Configuration
@PropertySource("classpath:druid-config.properties")
public class JdbcConfig {
    @Value("${druid.driverClassName}")
    private String driverClassName;
    @Value("${druid.url}")
    private String url;
    @Value("${druid.user}")
    private String username;
    @Value("${druid.password}")
    private String password;
    @Value("${druid.maxActive}")
    private int maxActive;

    @Bean
    public DataSource dataSource(){
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(driverClassName);
        ds.setUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        ds.setMaxActive(maxActive);
        ds.setMinIdle(0);
        return ds;
    }

    @Bean
    public SqlSessionFactoryBean sqlSessionFactory() throws IOException {
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        //将加载多个绝对匹配的所有Resource
        //将首先通过ClassLoader.getResource("META-INF")加载非模式路径部分
        //然后进行遍历模式匹配
        Resource[] resources =  resolver.getResources("classpath*:com/nd/mathlearning/server/*/dao/mapper/*.xml");
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource());
        bean.setMapperLocations(resources);
        bean.setConfigLocation(resolver.getResource("classpath:mybatis/mybatis-config.xml"));
        return bean;
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplate() throws Exception {
        SqlSessionTemplate bean = new SqlSessionTemplate(sqlSessionFactory().getObject());
        return bean;
    }
}
```
`@Configuration`表明此类用作Spring配置，`@PropertySource`说明读取哪个配置文件，`@Value`将`properties`中的配置项注入进变量，`@Bean`注解的方法表明实例化对象。还有需要注意的就是`ResourcePatternResolver`类的用法，是用来加载工程配置文件的。
