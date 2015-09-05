---
layout: post
title: Spring事件机制
date: 2015-09-03 14:37:28
comments: true
categories: java
---
在做业务开发的时候，我们经常遇到一种情况：随着需求的增加和完善，会不断的在同一个方法中增加业务逻辑。  
## 一个数据审核场景例子
例如一个数据的审核业务，原始需求为：  
>1. 提交审核
2. 保存数据，修改状态为已经审核  

第二版本的需求加入了新的归档状态，条件是审核时间小于最后完成时间，归档后进行统计计算：
>1. 提交审核
2. 保存数据，修改状态为已经审核  
3. 判断时间是否小于完成时间，若为是则进行归档操作

第三版系统增加了推送能力，审核和归档完成时分别推送消息给相关人员：
>1. 提交审核
2. 保存数据，修改状态为已经审核
3. 推送消息给审核相关人员
4. 判断时间是否小于完成时间，若为是则进行归档操作
5. 推送消息给归档相关人员


<!-- more -->

### 一种解决方式
最常想到的解决方法就是每次改变需求时都在`提交审核`方法里增加业务逻辑，伪代码例如：
```java
/**
 * 审核方法
 * @param id 数据id
 */
public void audit(String id){
  updateData(id); //保存业务数据并改变审核状态,对应原始需求
  notifyAudit();  //通知审核相关人员，对应第三版需求
  if(isArchive(id)){ //数据是否可以归档，对应第二版需求
    archive(id); //处理归档逻辑
    notifyArchive(); //通知归档相关人员，对应第三版需求
  }
}
```
每次改变需求的时候都要在审核方法里增加修改业务逻辑，增加了此方法的耦合性。而且像消息推送方法可以采用异步线程来实现，使主线程请求可以立即响应。

### 使用事件的方式解决
做过前端开发的同学对事件并不会陌生，某种条件下触发一个事件，每种事件都对应一种处理方式。  
对于这个业务场景我们可以定义审核为一种事件，归档也是一种事件。  
* 修改审核数据后触发审核事件
* 审核事件中发送推送消息，判断数据是否可以归档，是的话执行归档并触发归档事件
* 归档事件中发送推送消息

此种方式组织代码，以后的业务扩展就只要在对应事件中修改。  
并且事件的触发与执行可以使用异步线程实现，加大了系统吞吐量。  
![时序图1](/image/spring-event1.png)

另外，如果有其他的功能需要实现审核或归档功能，只需要触发对应的事件就行。

## Spring中事件机制
`Spring`本身在初始化的时候就用到了事件，我们在编写业务应用时也能很方便地定义自己的事件。  
### 定义事件
定义一个继承`ApplicationEvent`的类：  
```java
public class AuditEvent extends ApplicationEvent {  
    private String id;
    public AuditEvent(final Object source, String id) {  
        super(source);  
        this.id = id;
    }

    public String getId(){
        return id;
    }
}
```
事件中的参数可以由我们自己控制，这里构造方法第一个参数是触发事件的来源，第二个是事件中数据主键。

### 定义事件监听器
实现`ApplicationListener`接口，并使用泛型指定监听哪种事件：
```java
@Component  
public class AuditListener implements ApplicationListener<AuditEvent> {  
    @Override  
    public void onApplicationEvent(final AuditEvent event) {  
        System.out.println("触发了审核事件：" + event.getId());  
    }  
}  
```
注意监听器也要打上`@Component`标签，这样就被`Spring`纳入`bean`工厂统一管理起来。

### 触发事件
在容器中获得`ApplicationContext`，调用`publishEvent`即可。
```java
@Service
public class AuditService {  
    @Autowired  
    private ApplicationContext applicationContext;  

    public void audit(String id) {  
        applicationContext.publishEvent(new AuditEvent(this, id));  
    }  
}  
```

* 参考文章
[详解Spring事件驱动模型](http://jinnianshilongnian.iteye.com/blog/1902886)
