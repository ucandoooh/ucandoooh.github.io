---
title: Spring测试-Junit4和Junit5资源引入方式
date: 2021-05-31 22:57:11
tags:
    - Java
    - Spring
---

### 环境

* IDE - [IDEA](https://www.jetbrains.com/idea/)（Community）
* Framework - Spring Boot，Mybatis

***

### 前提

> 对于创建配置这块简略带过，主要阐述Junit4和Junit5的资源引入方式

1. <https://start.spring.io/>快速搭建引入Web和Mybatis依赖的Maven项目，导入IDEA，Ultimate版本自带构建Spring Boot。
2. [MVN仓库](https://mvnrepository.com/)查找依赖引入，引入Druid数据库连接池、MySql Connector、Junit
3. 创建Entity实体类、Dao数据库存取对象以及Mapper，Mybatis配置、Spring配置
4. 关键配置：spring-dao.xml中扫描Dao的配置

```xml
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <property name="basePackage" value="com.zxn.app.dao"/>
    </bean>
```

5. IDEA创建测试类的快捷键，<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>T</kbd>，或者鼠标悬浮在类上<kbd>Alt</kbd>+<kbd>Enter</kbd>选择create test

   ![create-new-test](create-new-test.jpg)

   选择Junit4/Junit5

   ![create-test-ui](create-test-ui.jpg)

---

### Junit4

```java
...
import static org.junit.Assert.*;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring/spring-dao.xml"})
public class SeckillDaoTest1 {

    @Autowired
    SeckillDao seckillDao;

    @Test
    public void queryById() {
        assertNotNull(seckillDao);
    }
}
```

1. @RunWith加载SpringJUnit4ClassRunner测试驱动，用于跑Spring测试
2. @ContextConfiguration加载资源文件，源文件路径下的spring/spring-dao.xml，用于注入Dao
3. 此处使用的断言是org.junit.Assert类的静态方法，判断是否成功加载spring-dao.xml，扫描注入了Dao，成功则Dao不为null
4. 测试通过不抛异常，并有log打印出加载资源的过程

---

### Junit5

```java
...
import static org.junit.jupiter.api.Assertions.*;

//@ExtendWith(SpringExtension.class)
//@ContextConfiguration(locations = {"classpath:spring/spring-dao.xml"})
@SpringJUnitConfig(locations = {"classpath:spring/spring-dao.xml"})
class SeckillDaoTest2 {

    @Autowired
    SeckillDao seckillDao;

    @Test
    void queryById() {
        assertNotNull(seckillDao);
    }
}
```

1. @ExtendWith(SpringExtension.class)加载Spring测试框架
2. @ContextConfiguration加载资源文件，源文件路径下的spring/spring-dao.xml，用于注入Dao

> 以上两者可以用@SpringJUnitConfig代替，简洁明了

3. 此处使用的断言是org.junit.jupiter.api.Assertions类的静态方法，判断是否成功加载spring-dao.xml，扫描注入了Dao，成功则Dao不为null
4. 测试通过不抛异常，并有log打印出加载资源的过程

#### 测试启动Web服务

```java
@ContextConfiguration(locations = {"classpath:spring/spring-dao.xml"})
@SpringBootTest(classes = AppApplication.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SeckillDaoTest2 {

    @Autowired
    SeckillDao seckillDao;

    @Test
    void queryById() {
        assertNotNull(seckillDao);
    }
}
```

1. 使用@SpringBootTest加载Web启动类AppApplication
2. webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT使用本地的一个随机端口启动服务
3. 配合使用@ContextConfiguration加载资源或者在AppApplication中加上@ImportResource

> 注意切勿重复加载资源，不然会报错

```java
@SpringBootApplication
//@ImportResource("classpath:spring/spring-dao.xml")
public class AppApplication {

	public static void main(String[] args) {
		SpringApplication.run(AppApplication.class, args);
	}

}
```

---

### 题外话

![project-structure](project-structure.jpg)

因为项目是分模块管理，如上图所示。

Dao放在persistent模块，Web启动类AppApplication放在web模块

当在persistent模块测试启动AppApplication，@ImportResource("classpath:spring/spring-dao.xml")加载的是persistent模块classpath下的spring/spring-dao.xml。

而当单独在web模块启动AppApplication来启动web会提示

**class path resource [spring/spring-dao.xml] cannot be opened because it does not exist**

改成@ImportResource("classpath*:spring/spring-dao.xml")就可以了

原因：

>1. 加载的资源，不在当前ClassLoader的路径里，那么用classpath:前缀是找不到的，这种情况下就需要使用classpath*:前缀
>
>2. classpath:与classpath*:的区别在于，前者只会从第一个classpath中加载，而 后者会从所有的classpath中加载
