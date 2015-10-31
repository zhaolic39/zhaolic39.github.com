title: Spring Restful Api 单元测试
comments: true
date: 2015-10-31 16:39:58
tags:
categories:
---

和任何做产品的公司一样，软件公司对其软件产品的质量也是十分看重。  
虽然任何公司都会有测试部门，但软件产品的质量不能完全靠QA同学们的加班来保障。软件质量从程序开发阶段就应该引起重视。  
单元测试便是在开发阶段中保证质量的一种重要方式。

<!--more-->
## 什么是单元测试
* 一个单元指的是应用程序中可测试的最小的一组源代码。
* 源代码中包含明确的输入和输出的每一个方法被认为是一个可测试的单元。
* 单元测试也就是在完成每个模块后都进行的测试。从确保每个模块没有问题，从而提高整体的程序质量。

## 单元测试的目的
* 是将应用程序的所有源代码，隔离成最小的可测试的单元，保证每个单元的正确性。
* 理想情况下，如果每个单元都能保证正确，就能保证应用程序整体相当程度的正确性。
* 单元测试也是一种特殊类型的文档，相对于书面的文档，测试脚本本身往往就是对被测试代码的实际的使用代码，对于帮助开发人员理解被测试单元的使用是相当有帮助的。

## Spring Restful Api 服务单元测试
就以现在流行的`Restful`服务来举一个单元测试例子，主要包含`Controller`与`Service`层。

### 测试工具，使用`junit`+`mockito`+`powermock`测试框架
* 项目新增测试框架的`pom`依赖：
```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.10</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-all</artifactId>
	<version>1.8.4</version>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>1.5.6</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito</artifactId>
    <version>1.5.6</version>
    <scope>test</scope>
</dependency>
```
### 创建单元测试基类
```java
@ContextConfiguration(classes = {WebConfig.class })
@RunWith(SpringJUnit4ClassRunner.class)  
public class BaseCtlCfg extends AbstractContextControllerTests  {
}
```

### Controller层单元测试
* 测试类继承`BaseCtlCfg`，加载、初始化`Spring`相关的配置。
* 在`@Before`中初始化`Mock`。
* 从`Spring`容器中获取`Controller`实例，通过反射机制用模拟的`service`类替换原有的`service`类。
* 根据测试的`Case`构建模拟数据。
* 通过`Mock`的`http method`请求方法调用接口，断言返回的`http status`、`body`是否符合期望。
```java
public class DraftControllerTest extends BaseCtlCfg{
    @Mock
    private  DraftService draftService;
    @Autowired
    protected WebApplicationContext wac;
    @Before
    public void before(){
        //初始化Mock
        MockitoAnnotations.initMocks(this);
        //获取上下文容器中的controller
        DraftController draftController = (DraftController) wac.getBean("draftController");
        //用mock的Service替换原有的service
        ReflectionTestUtils.setField(draftController, "draftService", draftService);
        }
    @Test
    public void publishTest() throws Exception {
        DraftPublishReq request = new DraftPublishReq();
        request.setPublishMode(1);
        request.setClassIds("495331469627");
        request.setPublishTime(System.currentTimeMillis());
        request.setPublishTime(System.currentTimeMillis() + 2*24*60*60*1000);

        String json = JSON.toJSONString(request);
        String uri = "/v0.3/drafts/00691d21-71f4-4115-89d0-a7168c77ef8b/actions/publish";

        //构建模拟数据。
        DraftInfo draftInfo = new DraftInfo();
        draftInfo.setIsdel(false);
        //模拟方法调用，当draftId=00691d21-71f4-4115-89d0-a7168c77ef8b时，return draftInfo对象
        when(draftService.get("00691d21-71f4-4115-89d0-a7168c77ef8b")).thenReturn(draftInfo);

        String resStr = mockMvc.perform(post(uri, "json")
                .characterEncoding("UTF-8")
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.valueOf(MockUtil.APPLICATION_JSON))
                .content(json.getBytes()))
                .andExpect(status().isOk())
                .andReturn().getResponse().getContentAsString();

        JSONArray jsonArray = JSON.parseArray(resStr);
        Assert.assertEquals(1, jsonArray.size());
    }
}
```

### Service层测试
* `Mock`类的静态方法需要引入注解`@RunWith(PowerMockRunner.class)`、`@PrepareForTest(LifecycleServiceV06.class)`（`LifecycleServiceV06.class`为Mock的静态方法所在的类）。
* 通过`@Spy`注解`Mock`测试类的真实对象。
* 在`@Before`中初始化`Mock`，并替换类中要`Mock`的属性。
* 构建模拟数据
* 方法如果有返回值，断言返回值是否符合期望。
```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(LifecycleServiceV06.class)
public class HomeworkBizServiceTest{

    @Spy //mock真实对象
    private HomeworkBizService homeworkBizService = new HomeworkBizService();
    @Mock
    private HomeworkExtendRepository homeworkExtendRepository;

    @Before
    public void before() {
        //初始化Mock
        MockitoAnnotations.initMocks(this);
        //用mock的repository替换原有的repository
        ReflectionTestUtils.setField(homeworkBizService, "homeworkExtendRepository", homeworkExtendRepository);
    }

    @Test
    public void createHomeworkPackage() throws Exception {
        Long userId = 2107222934l;
        UserInfo userInfo = new UserInfo();
        userInfo.setUserId(String.valueOf(userId));
        String homeworkId="8b310d5d-ebd2-4798-bff3-2863fe1e1cd7";
        //构建模拟数据
         HomeworkExtend extend = new HomeworkExtend();
        extend.setLcmsHomeworkId(homeworkId);
        LifecycleSession session = new LifecycleSession();
        //mock类静态方法
         PowerMockito.mockStatic(LifecycleServiceV06.class);
        //模拟方法调用返回
         PowerMockito.when(LifecycleServiceV06.uploadHomework(homeworkId, userId)).thenReturn(session);
        //模拟方法调用返回
         when(homeworkExtendRepository.get(homeworkId)).thenReturn(extend);
        //void方法do nothing
        doNothing().when(homeworkBizService).createHomeworkPackage(homeworkId,session);

        homeworkBizService.createHomeworkPackage(homeworkId, userInfo);
    }
}
```
