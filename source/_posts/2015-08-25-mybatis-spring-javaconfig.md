---
layout: post
title: Spring下的Mybatis配置
date: 2015-08-19 00:05:58
comments: true
categories: java
---
`J2EE`开发界中一个永远避免不了的争论就是`Hibernate`与`Mybatis`哪个好。虽然我自己倾向`Hibernate`，但这两种`ORM`框架我也都没有在比较大型的场景上使用过，没有资格做评价。

最近公司有一个项目，主程让我用`Spring`与`Mybatis`搭建服务端框架，这里就做一下简单的学习记录。
<!-- more -->
## Java Config方式的Mybatis启动配置
首先在`maven`中加上`Mybatis`与`mybatis-spring`的依赖：
```xml
<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis-spring</artifactId>
   <version>1.2.2</version>
</dependency>

<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis</artifactId>
   <version>3.2.8</version>
</dependency>
```
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
        //加载匹配到的xml映射配置
        Resource[] resources =  resolver.getResources("classpath*:com/nd/mathlearning/server/*/dao/mapper/*.xml");
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(dataSource());
        bean.setMapperLocations(resources);
        //加载Mybatis配置
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
* `@Configuration`表明此类用作`Spring配置`
* `@PropertySource`说明读取哪个配置文件
* `@Value`将`properties`中的配置项注入进变量
* `@Bean`注解的方法表明实例化对象。  
* `ResourcePatternResolver`是用来加载工程配置文件。

## Mybatis的xml映射文件
与`Hibernate`类似，`Mybatis`需要写`xml`配置来实现`Java`类与数据库表之间的映射关系，更多的，`Mybatis`框架执行的所有`sql`也都配置在`xml`中。
来看一段完整的xml配置：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.nd.mathlearning.server.personal.dao.UcUseridMapMapper" >
  <resultMap id="BaseResultMap" type="com.nd.mathlearning.server.personal.dto.UcUseridMapEntity" >
    <id column="uc_user_id" property="ucUserId" jdbcType="BIGINT" />
    <result column="user_id" property="userId" jdbcType="BIGINT" />
    <result column="user_name" property="userName" jdbcType="VARCHAR" />
  </resultMap>
  <sql id="Base_Column_List" >
    uc_user_id, user_id, user_name
  </sql>
  <select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="java.lang.Long" >
    select
    <include refid="Base_Column_List" />
    from uc_userid_map
    where uc_user_id = #{ucUserId,jdbcType=BIGINT}
  </select>
  <delete id="deleteByPrimaryKey" parameterType="java.lang.Long" >
    delete from uc_userid_map
    where uc_user_id = #{ucUserId,jdbcType=BIGINT}
  </delete>
  <insert id="insert" parameterType="com.nd.mathlearning.server.personal.dto.UcUseridMapEntity" >
    insert into uc_userid_map (uc_user_id, user_id, user_name
      )
    values (#{ucUserId,jdbcType=BIGINT}, #{userId,jdbcType=BIGINT}, #{userName,jdbcType=VARCHAR}
      )
  </insert>
  <update id="updateByPrimaryKeySelective" parameterType="com.nd.mathlearning.server.personal.dto.UcUseridMapEntity" >
    update uc_userid_map
    <set >
      <if test="userId != null" >
        user_id = #{userId,jdbcType=BIGINT},
      </if>
      <if test="userName != null" >
        user_name = #{userName,jdbcType=VARCHAR},
      </if>
    </set>
    where uc_user_id = #{ucUserId,jdbcType=BIGINT}
  </update>
</mapper>
```
* 根节点`mapper`中的`namespace`声明了此配置的唯一编号，*在此后的`java`的`api`调用就需要用到此属性*。
* `resultMap`定义表到`java`实体类之间的字段映射关系
* `column`是表字段
* `property`是类属性
* `jdbcType`是表字段类型。  
* `sql`节点在这里定义了一个`sql`查询片段，使用`include`语法拼接，用于简化配置。  
* `select`，`insert`，`update`，`delete`节点定义四种`sql`操作的方法。`id`定义方法的标识；
* `parameterType`说明方法的参数类型，可以是`javabean`，但一个方法只能有一个输入参数。

更详细的`mapper`配置说明参考官网：
* [Mapper XML 文件](http://mybatis.github.io/mybatis-3/zh/sqlmap-xml.html)

### mapper与实体生成工具
既然`Mybatis`所有的逻辑操作都是配置在`xml`中，那就把原来在`java`中的`sql`开发工作转换到了`xml`上。  
与`Hibernate`一样，`Mybatis`提供了一个简易的逆向工程工具，帮助我们根据已有数据表生成对应的`javabean`和基本操作`xml`配置。  
在`maven`中增加生成器依赖：
```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.3.2</version>
</dependency>
```
编写生成配置`xml`
```xml
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>
	<!-- classPathEntry:数据库的JDBC驱动 -->
	<classPathEntry location="F:\mylib\mysql-connector-java-5.1.30.jar" />

	<!--生成映射的类型，也可以生成ibatis的。具体参看mybatis-generator -->
	<context id="DB2Tables3" targetRuntime="MyBatis3">
		<plugin type="org.mybatis.generator.plugins.CaseInsensitiveLikePlugin"></plugin>
		<plugin type="org.mybatis.generator.plugins.SerializablePlugin"></plugin>
		<!-- 去除自动生成的注释 -->
		<commentGenerator>
			<property name="suppressDate" value="true" />
			<property name="suppressAllComments" value="true" />
		</commentGenerator>
		<!-- 数据库配置 -->
		<jdbcConnection driverClass="com.mysql.jdbc.Driver"
			connectionURL="jdbc:mysql://172.24.133.201:3306/mylearning" userId="user_account"
			password="user_account">
		</jdbcConnection>
		<javaTypeResolver>
			<property name="forceBigDecimals" value="false" />
		</javaTypeResolver>
		<!--以下三个标签主要解析targetPackage和targetProject。其它的具体参看mybatis-generator -->
		<!-- targetProject:自动生成代码的位置 -->
		<javaModelGenerator targetPackage="com.nd.mathlearning.server.microclass.dto" targetProject="src/main/java/">
			<property name="enableSubPackages" value="true" />
			<property name="trimStrings" value="true" />
		</javaModelGenerator>
		<sqlMapGenerator targetPackage="com.nd.mathlearning.server.microclass.dao" targetProject="src/main/java/">
			<property name="enableSubPackages" value="true" />
		</sqlMapGenerator>
    <!-- 要生成的表配置 -->
		<table tableName="mc_note" domainObjectName="McNoteEntity" enableCountByExample="false" enableUpdateByExample="false"
			   enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"></table>
	</context>
</generatorConfiguration>
```
有了生成配置`xml`后就可以调用`api`生成实体`javabean`，`dao`和`mapper`：
```java
public class MyBatisGenerator {
    public static void main(String args[]){
        String config ="";
        try {
            config = MyBatisGenerator.class.getResource("generator.xml").toURI().getPath();
        } catch (URISyntaxException e) {
            e.printStackTrace();
        }
        String[] arg = { "-configfile", config, "-overwrite" };
        ShellRunner.main(arg);
    }
}
```
参考文章：
* [使用Mybatis-Generator自动生成Dao、Model、Mapping相关文件](http://www.cnblogs.com/smileberry/p/4145872.html)

## 与Spring整合
在`j2ee`的各种开源框架中，我们最关心的可能就是怎么与`Spring`做整合。
### 数据映射器 MapperFactoryBean
`Mybatis`提供了一种简易的整合方式，使用`class`的方式定义`mapper`，而不需要配置`xml`。  
例如定义一个`interface`的`mapper`：
```java
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = #{userId}")
    User getUser(@Param("userId") long id);
}
```
然后在`spring`中定义：
```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
      <property name="mapperInterface" value="com.xxt.ibatis.dbcp.dao.UserMapper" />
      <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```
`Spring`容器中就产生了`userMapper`实例可以来执行我们想要的数据操作。这种方式有点类似`spring data`中`CrudRepository`的`@Query`用法。

### 抽象类SqlSessionDaoSupport的整合方式
从前面的启动配置我们知道了，`Mybatis`的`api`入口是`sqlSessionFactory`。`Mybatis`提供`SqlSessionDaoSupport`抽象类来方便获取`sqlSession`。  
先看一个UserDao的实现：
```java
public class UserDao extends SqlSessionDaoSupport{  
  @Autowired
  public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
    super.setSqlSessionTemplate(sqlSessionTemplate);
  }

  public User getUserById(User user) {  
     return (User) getSqlSession().selectOne("com.xxt.ibatis.dbcp.domain.User.getUser", user);  
  }  
}  
```
继承`SqlSessionDaoSupport`可以获得`getSqlSession`方法来得到`sqlSession`。然后通过`sqlSession`的`api`最终调用到`mapper`里配置的`sql`程序。  
sqlSession几个常用的api：
>```java
<T> T selectOne(String statement, Object parameter)  
<E> List<E> selectList(String statement, Object parameter)  
<K,V> Map<K,V> selectMap(String statement, Object parameter, String mapKey)  
int insert(String statement, Object parameter)  
int update(String statement, Object parameter)  
int delete(String statement, Object parameter)
```

第一个参数表示mapper中方法的唯一标识，结构为：*mapper标识* + . + *方法标识*。  
### mybatis-spring 1.2版本中SqlSessionDaoSupport的变化
首先看1.2版本中`SqlSessionDaoSupport`的注释信息：
>```java
/**
 * Convenient super class for MyBatis SqlSession data access objects.
 * It gives you access to the template which can then be used to execute SQL methods.
 * <p>
 * This class needs a SqlSessionTemplate or a SqlSessionFactory.
 * If both are set the SqlSessionFactory will be ignored.
 * <p>
 * {code Autowired} was removed from setSqlSessionTemplate and setSqlSessionFactory
 * in version 1.2.0.
 *
 * @author Putthibong Boonbong
 *
 * @see #setSqlSessionFactory
 * @see #setSqlSessionTemplate
 * @see SqlSessionTemplate
 * @version $Id$
 */
 ```

之前版本中setSqlSessionTemplate和setSqlSessionFactory是Autowired，1.2之后需要我们手动注入。  
这个改动应该是为了支持一个项目中建立多数据源的场景，我们可以用SqlSessionDaoSupport创建多个DAO层基类，选择不同的数据源注入。
### Mybatis插件配置



参考文章：
* [spring与mybatis三种整合方法](http://nirvana1988.iteye.com/blog/971246)
* [Java API sqlSeesion](http://mybatis.github.io/mybatis-3/zh/java-api.html#sqlSessions)
