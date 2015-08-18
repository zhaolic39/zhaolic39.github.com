---
layout: post
title: Servlet3.0特性下Spring配置
date: 2015-08-19 00:05:58
comments: true
categories: java
---
`Servlet3.0`环境下，提供给开发者一种基于`class`注解的`web`项目的配置方式，作为`web.xml`的补充或者替代，使得工程配置有更强的可读性：
```java
public class DefaultWebApplicationInitializer implements
        WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext appContext) throws ServletException {
        AnnotationConfigWebApplicationContext rootContext = new AnnotationConfigWebApplicationContext();
        rootContext.register(DefaultAppConfig.class);
        appContext.addListener(new ContextLoaderListener(rootContext));
        //注册dispatcher servlet
        ServletRegistration.Dynamic dispatcher = appContext.addServlet("dispatcher", new DispatcherServlet(rootContext));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");
    }
}
```
这个类相当于原来的`web.xml`配置文件。
## Spring的class配置
`Spring`及其生态圈也陆续支持此种方式的启动和配置。
`Spring`支持配置写成代码的形式以替代传统的`xml`配置，例如：
```java
@Configuration
@ComponentScan(basePackages = "com.nd.myproject")
public class DefaultAppConfig {
}
```
`@Configuration`表明此为`Spring`配置类，`@ComponentScan`说明了`Bean`扫描路径。

* 常用注解说明：
>`@Configuration` ： 类似于`spring`配置文件，负责注册`bean`，对应的提供了`@Bean`注解。
`@ComponentScan` ： 注解类查找规则定义 `<context:component-scan/>`
`@EnableAspectJAutoProxy` ： 激活`Aspect`自动代理 `<aop:aspectj-autoproxy/>`
`@Import` `@ImportResource`: 关联其它`Spring`配置  `<import resource="" />`
`@EnableCaching` ：启用缓存注解  `<cache:annotation-driven/>`
`@EnableTransactionManagement` ： 启用注解式事务管理 `<tx:annotation-driven />`
`@EnableWebMvcSecurity` ： 启用`SpringSecurity`安全验证

* 参考资料：  
[springmvc基于java config的实现](http://blog.csdn.net/xiejx618/article/details/42471135)
