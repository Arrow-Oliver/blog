---
title: "初学Spring"
date: 2023-03-31T22:38:30+08:00
draft: false
categories: [Java,笔记,Spring]
tags: [Spring]
card: false
weight: 0
---

# Spring简介

​	 Spring是一个开源框架，它由[Rod Johnson](https://baike.baidu.com/item/Rod Johnson)创建。它是为了解决企业应用开发的复杂性而创建的。 

​	 目前是JavaEE开发的灵魂框架。他可以简化JavaEE开发，可以非常方便整合其他框架，无侵入的进行功能增强。

​	 Spring的核心就是 控制反转(IoC)和面向切面(AOP) 。

# 1.IOC控制反转

## 概念

控制反转，之前对象的控制权在类手上，现在反转后到了Spring手上。

​	

## 入门案例

## ①导入依赖

导入SpringIOC相关依赖

~~~~xml
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.1.9.RELEASE</version>
        </dependency>
~~~~

## ②编写配置文件

在resources目录下创建applicationContext.xml文件，文件名可以任意取。但是建议叫applicationContext。

内容如下：

~~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--
        classs:配置类的全类名
        id:配置一个唯一标识
    -->
    <bean class="com.sangeng.dao.impl.StudentDaoImpl" id="studentDao"  >
    </bean>


</beans>
~~~~



## ③创建容器从容器中获取对象并测试

~~~~java
    public static void main(String[] args) {

//        1.获取StudentDaoImpl对象
        //创建Spring容器，指定要读取的配置文件路径
        ApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml");
        //从容器中获取对象
        StudentDao studentDao = (StudentDao) app.getBean("studentDao");
        //调用对象的方法进行测试
        System.out.println(studentDao.getStudentById(1));
    }
~~~~



## Bean的常用属性配置

### id 

​	bean的唯一标识，同一个Spring容器中不允许重复

### class

​	全类名，用于反射创建对象

### scope 

​	scope主要有两个值：singleton和prototype

​	如果设置为singleton则一个容器中只会有这个一个bean对象。默认容器创建的时候就会创建该对象。

​	如果设置为prototype则一个容器中会有多个该bean对象。每次调用getBean方法获取时都会创建一个新对象。



## DI依赖注入

​	依赖注入可以理解成IoC的一种应用场景，反转的是对象间依赖关系维护权。

​	

### set方法注入

在要注入属性的bean标签中进行配置。<u>前提是该类有提供属性对应的set方法。</u>

~~~~xml
    <bean class="com.sangeng.domain.Dog" id="dog">
        <property name="name" value="小白"></property>
        <property name="age" value="6"></property>
    </bean>

    <bean class="com.sangeng.domain.Student" id="student" >
        <!--
            name属性用来指定要设置哪个属性
            value属性用来设置要设置的值
            ref属性用来给引用类型的属性设置值，可以写上Spring容器中bean的id
        -->
        <property name="name" value="东南枝"></property>
        <property name="age" value="20"></property>
        <property name="id" value="1"></property>
        <property name="dog" ref="dog"></property>
    </bean>
~~~~



### 有参构造注入

在要注入属性的bean标签中进行配置。<u>前提是该类有提供对应的有参构造。</u>

~~~~xml
    <!--使用有参构造进行注入-->
    <bean class="com.sangeng.domain.Student" id="student2" >
        <constructor-arg name="name" value="自挂东南枝"></constructor-arg>
        <constructor-arg name="age" value="20"></constructor-arg>
        <constructor-arg name="id" value="30"></constructor-arg>
        <constructor-arg name="dog" ref="dog"></constructor-arg>
    </bean>
~~~~



### 复杂类型属性注入

实体类如下：

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private int age;
    private String name;
    private Phone phone;
    private List<String> list;
    private List<Phone> phones;
    private Set<String> set;
    private Map<String, Phone> map;
    private int[] arr;
    private Properties properties;
}
~~~~

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Phone {
    private double price;
    private String name;
    private String password;
    private String path;

}
~~~~



配置如下：

~~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="com.sangeng.domain.Phone" id="phone">
        <property name="price" value="3999"></property>
        <property name="name" value="黑米"></property>
        <property name="password" value="123"></property>
        <property name="path" value="qqqq"></property>
    </bean>
    
    <bean class="com.sangeng.domain.User" id="user">
        <property name="age" value="10"></property>
        <property name="name" value="大队长"></property>
        <property name="phone" ref="phone"></property>
        <property name="list">
            <list>
                <value>三更</value>
                <value>西施</value>
            </list>
        </property>

        <property name="phones">
            <list>
                <ref bean="phone"></ref>
            </list>
        </property>

        <property name="set">
            <set>
                <value>setEle1</value>
                <value>setEle2</value>
            </set>
        </property>

        <property name="map">
            <map>
                <entry key="k1" value-ref="phone"></entry>
                <entry key="k2" value-ref="phone"></entry>
            </map>
        </property>

        <property name="arr">
            <array>
                <value>10</value>
                <value>11</value>
            </array>
        </property>

        <property name="properties">
            <props>
                <prop key="k1">v1</prop>
                <prop key="k2">v2</prop>
            </props>
        </property>
    </bean>
</beans>
~~~~

## SPEL

​	我们可以再配置文件中使用SPEL表达式。写法如下:

~~~~xml
        <property name="age" value="#{20}"/>
        <property name="car" value="#{car}"/>
~~~~

​	注意：SPEL需要写到value属性中，不能写到ref属性。



## 配置文件

### 读取properties文件

​	我们可以让Spring读取properties文件中的key/value，然后使用其中的值。

### ①设置读取properties

在Spring配置文件中加入如下标签：指定要读取的文件的路径。

~~~~xml
<context:property-placeholder location="classpath:filename.properties">
~~~~

其中的classpath表示类加载路径下。

我们也会用到如下写法：classpath:**.properties  其中的*  * 表示文件名任意。

**注意：context命名空间的引入是否正确**

### ②使用配置文件中的值

在我们需要使用的时候可以使用${key}来表示具体的值。注意要再value属性中使用才可以。例如：

~~~~xml
<property name="propertyName" value="${key}"/>
~~~~



### 引入Spring配置文件

​	我们可以在主的配置文件中通过import标签的resource属性，引入其他的xml配置文件

~~~~xml
<import resource="classpath:applicationContext-book.xml"/>
~~~~



## 低频知识点

### bean的配置

#### name属性

​	我们可以用name属性来给bean取名。例如：

~~~~xml
    <bean class="com.alibaba.druid.pool.DruidDataSource" id="dataSource" name="dataSource2,dataSource3">
        <property name="driverClassName" value="${jdbc.driver}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
    </bean>
~~~~

​	获取的时候就可以使用这个名字来获取了

~~~~java
    public static void main(String[] args) {

        ApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml");
        DruidDataSource dataSource = (DruidDataSource) app.getBean("dataSource3");
        System.out.println(dataSource);

    }
~~~~



#### lazy-init

​	可以控制bean的创建时间，如果设置为true就是在第一次获取该对象的时候才去创建。

~~~~xml
    <bean class="com.alibaba.druid.pool.DruidDataSource" lazy-init="true"  id="dataSource" name="dataSource2,dataSource3">
        <property name="driverClassName" value="${jdbc.driver}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
    </bean>
~~~~



#### init-method

​	可以用来设置初始化方法，设置完后容器创建完对象就会自动帮我们调用对应的方法。

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Student {

    private String name;
    private int id;
    private int age;
	//初始化方法
    public void init(){
        System.out.println("对学生对象进行初始化操作");
    }

}

~~~~

~~~~xml
<bean class="com.sangeng.domain.Student" id="student" init-method="init"></bean>
~~~~

**注意：配置的初始化方法只能是空参的。**



#### destroy-method

​	可以用来设置销毁之前调用的方法，设置完后容器销毁对象前就会自动帮我们调用对应的方法。

~~~~xml
    <bean class="com.sangeng.domain.Student" id="student"  destroy-method="close"></bean>
~~~~

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Student {

    private String name;
    private int id;
    private int age;

    public void init(){
        System.out.println("对学生对象进行初始化操作");
    }

    public void close(){
        System.out.println("对象销毁之前调用，用于释放资源");
    }
}

~~~~

**注意：配置的方法只能是空参的。**



#### factory-bean&factory-method

​	当我们需要让Spring容器使用工厂类来创建对象放入Spring容器的时候可以使用factory-bean和factory-method属性。



**配置实例工厂创建对象**

配置文件中进行配置

~~~~xml
    <!--创建实例工厂-->
    <bean class="com.sangeng.factory.CarFactory" id="carFactory"></bean>
    <!--使用实例工厂创建Car放入容器-->
    <!--factory-bean 用来指定使用哪个工厂对象-->
    <!--factory-method 用来指定使用哪个工厂方法-->
    <bean factory-bean="carFactory" factory-method="getCar" id="car"></bean>
~~~~

创建容器获取对象测试

~~~~java
        ClassPathXmlApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml");
        //获取car对象
        Car c = (Car) app.getBean("car");
        System.out.println(c);
~~~~



**配置静态工厂创建对象**

配置文件中进行配置

~~~~xml
    <!--使用静态工厂创建Car放入容器-->
    <bean class="com.sangeng.factory.CarStaticFactory" factory-method="getCar" id="car2"></bean>
~~~~

创建容器获取对象测试

~~~~java
        ClassPathXmlApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml");
        //获取car对象
        Car c = (Car) app.getBean("car2");
        System.out.println(c);
~~~~

# 2.注解开发

为了简化配置，Spring支持使用注解代替xml配置。

## 注解开发准备工作

​	如果要使用注解开发必须要开启组件扫描，这样加了注解的类才会被识别出来。Spring才能去解析其中的注解。

~~~~xml
<!--启动组件扫描，指定对应扫描的包路径，该包及其子包下所有的类都会被扫描，加载包含指定注解的类-->
<context:component-scan base-package="com.sangeng"/>
~~~~

## IOC相关注解

**@Component,@Controller,@Service ,@Repository**	

​	上述4个注解都是加到类上的。

​	他们都可以起到类似bean标签的作用。可以把加了该注解类的对象放入Spring容器中。

​	实际再使用时选择任意一个都可以。但是后3个注解是语义化注解。

​	如果是Service类要求使用@Service。

​	如果是Dao类要求使用@Repository

​	如果是Controllerl类(SpringMVC中会学习到)要求使用@Controller

​	如果是其他类可以使用@Component



例如：

配置文件如下：

~~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
<!--启动组件扫描，指定对应扫描的包路径，该包及其子包下所有的类都会被扫描，加载包含指定注解的类-->
    <context:component-scan base-package="com.sangeng"></context:component-scan>

</beans>
~~~~

类如下：

~~~~java
@Repository("userDao")
public class UserDaoImpl implements UserDao {

    public void show() {
        System.out.println("查询数据库，展示查询到的数据");
    }
}

~~~~

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Component("phone")
public class Phone {
    private double price;
    private String name;
    private String password;
    private String path;

}

~~~~

~~~~java
@Service("userService")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserServiceImpl implements UserService {


    private UserDao userDao;

    private int num;

    private String str;


    public void show() {
        userDao.show();
    }
}

~~~~



测试类如下：

~~~~java
public class Demo {
    public static void main(String[] args) {
        //创建容器
        ClassPathXmlApplicationContext app = new ClassPathXmlApplicationContext("applicationContext.xml");
        //获取对象
        UserDao userDao = (UserDao) app.getBean("userDao");
        Phone phone = (Phone) app.getBean("phone");
        UserService userService = (UserService) app.getBean("userService");
        System.out.println(phone);
        System.out.println(userService);
        System.out.println(userDao);
    }
}
~~~~



## DI相关注解

​	如果一个bean已经放入Spring容器中了。那么我们可以使用下列注解实现属性注入，让Spring容器帮我们完成属性的赋值。



### @Value

​	主要用于String,Integer等可以直接赋值的属性注入。不依赖setter方法，支持SpEL表达式。

例如：

~~~~java
@Service("userService")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserServiceImpl implements UserService {
    private UserDao userDao;
    @Value("199")
    private int num;
    @Value("三更草堂")
    private String str;
    @Value("#{19+3}")
    private Integer age;


    public void show() {
        userDao.show();
    }
}
~~~~



### @AutoWired

​	Spring会给加了该注解的属性自动注入数据类型相同的对象。

例如：

~~~~java
@Service("userService")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;

    @Value("199")
    private int num;
    @Value("三更草堂")
    private String str;

    @Value("#{19+3}")
    private Integer age;


    public void show() {
        userDao.show();
    }
}

~~~~



​	**required属性代表这个属性是否是必须的，默认值为true。如果是true的话Spring容器中如果找不到相同类型的对象完成属性注入就会出现异常。**





### @Qualifier

​	<u>如果相同类型的bean在容器中有多个时</u>，单独使用@AutoWired就不能满足要求，这时候可以再加上@Qualifier来指定bean的名字从容器中获取bean注入。

例如：

~~~~java
    @Autowired
    @Qualifier("userDao2")
    private UserDao userDao;
~~~~



**注意：该直接不能单独使用。单独使用没有作用**



## xml配置文件相关注解

### @Configuration

​	**标注在类上，表示当前类是一个配置类。我们可以用注解类来完全替换掉xml配置文件。**

​	注意：如果使用配置类替换了xml配置，spring容器要使用：AnnotationConfigApplicationContext

例如：

~~~~java
@Configuration
public class ApplicationConfig {
}

~~~~



### @ComponentScan

​	**可以用来代替context:component-scan标签来配置组件扫描。**

​	basePackages属性来指定要扫描的包。

​	注意要加在配置类上。

例如：

~~~~java
@Configuration
@ComponentScan(basePackages = "com.sangeng")//指定要扫描的包
public class ApplicationConfig {
}

~~~~





### @Bean

​	**可以用来代替bean标签，主要用于第三方类的注入。**

​	使用：定义一个方法，在方法中创建对应的对象并且作为返回值返回。然后在方法上加上@Bean注解，注解的value属性来设置bean的名称。

例如：

~~~~java
@Configuration
@ComponentScan(basePackages = "com.sangeng")
public class ApplicationConfig {

    @Bean("dataSource")
    public DruidDataSource getDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName("com.mysql.jdbc.Driver");
        druidDataSource.setUsername("root");
        druidDataSource.setUrl("jdbc:mysql://localhost:3306/mybatis_db");
        druidDataSource.setPassword("root");
        return druidDataSource;
    }

}
~~~~



**注意事项：如果同一种类型的对象在容器中只有一个，我们可以不设置bean的名称。**

具体写法如下：

~~~~java
@Configuration
@ComponentScan(basePackages = "com.sangeng")
public class ApplicationConfig {

    @Bean
    public DruidDataSource getDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName("com.mysql.jdbc.Driver");
        druidDataSource.setUsername("root");
        druidDataSource.setUrl("jdbc:mysql://localhost:3306/mybatis_db");
        druidDataSource.setPassword("root");
        return druidDataSource;
    }

}
~~~~

获取方式如下：

~~~~java
    public static void main(String[] args) {
        //创建注解容器
        AnnotationConfigApplicationContext app = new AnnotationConfigApplicationContext(ApplicationConfig.class);
		//根据对应类的字节码对象获取
        DataSource bean = app.getBean(DataSource.class);
        System.out.println(userService);
    }
~~~~





#### @PropertySource

​	**可以用来代替context:property-placeholder，让Spring读取指定的properties文件。然后可以使用@Value来获取读取到的值。**



​	**使用：在配置类上加@PropertySource注解，注解的value属性来设置properties文件的路径。**

​	**然后在配置类中定义成员变量。在成员变量上使用@Value注解来获取读到的值并给对应的成员变量赋值。**



例如：

~~~~properties
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis_db
jdbc.username=root
jdbc.password=root
~~~~

读取文件并且获取值

~~~~java
@Configuration
@ComponentScan(basePackages = "com.sangeng")
@PropertySource("jdbc.properties")
public class ApplicationConfig {

    @Value("${jdbc.driver}")
    private String driverClassName;
    @Value("${jdbc.url}")
    private String url;
    @Value("${jdbc.username}")
    private String username;
    @Value("${jdbc.password}")
    private String password;


    @Bean
    public DruidDataSource getDataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName(driverClassName);
        druidDataSource.setUsername(username);
        druidDataSource.setUrl(url);
        druidDataSource.setPassword(password);
        return druidDataSource;
    }

}

~~~~



**注意事项：使用@Value获取读到的properties文件中的值时使用的是${key},而不是#{key}。**



## 如何选择

①SSM  

​		自己项目中的类的IOC和DI都使用注解，对第三方jar包中的类，配置组件扫描时使用xml进行配置。

②SpringBoot

​		纯注解开发

# 3.AOP面向切面编程

## 概念 

​	 AOP为Aspect Oriented Programming的缩写，意为：面向切面编程。**他是一种可以在不修改原来的核心代码的情况下给程序动态统一进行增强的一种技术。** 

## 入门案例

### ①添加依赖

需要添加SpringIOC相关依赖和AOP相关依赖。

~~~~xml
        <!--SpringIOC相关依赖-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.1.9.RELEASE</version>
        </dependency>
        <!--AOP相关依赖-->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.8.13</version>
        </dependency>
~~~~

### ②相关bean要注入容器中

加@Service注解

~~~~java
@Service
public class PhoneService {

    public void deleteAll(){
        System.out.println("PhoneService中deleteAll的核心代码");
    }
}

~~~~

~~~~java
@Service
public class UserService {


    public void deleteAll(){
        System.out.println("UserService中deleteAll的核心代码");
    }
}

~~~~



### 实现AOP

### ①开启AOP注解支持 

使用**aop:aspectj-autoproxy**标签

~~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <!--开启组件扫描-->
    <context:component-scan base-package="com.sangeng"></context:component-scan>
    <!--开启aop注解支持-->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>

</beans>
~~~~



### ②创建切面类

创建一个类，在类上加上@Component和@Aspect

使用@Pointcut注解来指定要被增强的方法

使用@Before注解来给我们的增强代码所在的方法进行标识，并且指定了增强代码是在被增强方法执行之前执行的。

~~~~java
@Component
@Aspect
public class MyAspect {

//    用Pointcut注解中的属性来指定对哪些方法进行增强
    @Pointcut("execution(* com.sangeng.service.*.*(..))")
    public void pt(){}

    /*
        用@Before注解来指定该方法中是增强的代码，并且是在被增强方法执行前执行的
        @Before的属性写上加了@Pointcut注解的方法: 方法名()
    */
    @Before("pt()")
    public void methodbefore(){
        System.out.println("方法被调用了");
    }

}
~~~~



### ③测试

~~~~java
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        PhoneService phoneService = applicationContext.getBean(PhoneService.class);
        UserService userService = applicationContext.getBean(UserService.class);
        phoneService.deleteAll();

    }
~~~~



## AOP核心概念

-  Joinpoint（连接点）：所谓连接点是指那些可以被增强到的点。在spring中,这些点指的是方法，因为spring只支持方法类型的连接点

-  **Pointcut（切入点）：所谓切入点是指被增强的连接点（方法）**

-  **Advice（通知/ 增强）：所谓通知是指具体增强的代码**

-  Target（目标对象）：被增强的对象就是目标对象

-  **Aspect（切面）：是切入点和通知（引介）的结合**

-  Proxy （代理）：一个类被 AOP 增强后，就产生一个结果代理类



### 切点表达式

​		可以使用切点表达式来表示要对哪些方法进行增强。



写法：**execution([修饰符] 返回值类型 包名.类名.方法名(参数))**

- 访问修饰符可以省略，大部分情况下省略
- 返回值类型、包名、类名、方法名可以使用星号*  代表任意
- 包名与类名之间一个点 . 代表当前包下的类，两个点 .. 表示当前包及其子包下的类
- 参数列表可以使用两个点 .. 表示任意个数，任意类型的参数列表



例如：

```java
execution(* com.sangeng.service.*.*(..))   表示com.sangeng.service包下任意类，方法名任意，参数列表任意，返回值类型任意
   
execution(* com.sangeng.service..*.*(..))   表示com.sangeng.service包及其子包下任意类，方法名任意，参数列表任意，返回值类型任意
    
execution(* com.sangeng.service.*.*())     表示com.sangeng.service包下任意类，方法名任意，要求方法不能有参数，返回值类型任意
    
execution(* com.sangeng.service.*.delete*(..))     表示com.sangeng.service包下任意类，要求方法不能有参数，返回值类型任意,方法名要求已delete开头
```





**切点函数@annotation**

​	我们也可以在要增强的方法上加上注解。然后使用@annotation来表示对加了什么注解的方法进行增强。



写法：**@annotation(注解的全类名)**

例如：

定义注解如下

~~~~java
@Target({ElementType.METHOD})//该注解可以加在方法上
@Retention(RetentionPolicy.RUNTIME)
public @interface InvokeLog {
}
~~~~

给需要增强的方法增加注解

~~~~java
@Service
public class PhoneService {

    @InvokeLog  
    public void deleteAll(){
        System.out.println("PhoneService中deleteAll的核心代码");
    }
}
~~~~

切面类中使用@annotation来确定要增强的方法

~~~~java
@Component
@Aspect
public class MyAspect {

//    用Pointcut注解中的属性来指定对哪些方法进行增强
    @Pointcut("@annotation(com.sangeng.aspect.InvokeLog)")
    public void pt(){}

    /*
        用@Before注解来指定该方法中是增强的代码，并且是在被增强方法执行前执行的
        @Before的属性写上加了@Pointcut注解的方法: 方法名()
    */
    @Before("pt()")
    public void methodbefore(){
        System.out.println("方法被调用了");
    }
}

~~~~





### 通知分类

- @Before：前置通知,在目标方法执行前执行
- @AfterReturning： 返回后通知，在目标方法执行后执行，如果出现异常不会执行
- @After：后置通知，在目标方法之后执行，无论是否出现异常都会执行 
- @AfterThrowing：异常通知，在目标方法抛出异常后执行

- **@Around：环绕通知，围绕着目标方法执行**



理解不同通知执行时机。（**下面的伪代码是用来理解单个通知的执行时机的，不能用来理解多个通知情况下的执行顺序。如果需要配置多个通知我们会选择使用Around通知，更加的清晰并且好用**）

~~~~java
    public Object test() {
        before();//@Before 前置通知
        try {
            Object ret = 目标方法();//目标方法调用
            afterReturing();//@AfterReturning 返回后通知
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            afterThrowing();//@AfterThrowing 异常通知通知
        }finally {
            after();//@After 后置通知
        }
        return ret;
    }
~~~~



环绕通知非常特殊，它可以对目标方法进行全方位的增强。

例如：

~~~~java
  	@Around("pt()")
    public void around(ProceedingJoinPoint pjp){
        System.out.println("目标方法前");
        try {
            pjp.proceed();//目标方法执行
            System.out.println("目标方法后");
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            System.out.println("目标方法出现异常");
        }finally {
            System.out.println("finally中进行增强");
        }
    }
~~~~

### 获取被增强方法相关信息

​	我们实际对方法进行增强时往往还需要获取到被增强代码的相关信息，比如方法名，参数，返回值，异常对象等。



​	我们可以在除了环绕通知外的所有通知方法中增加一个**JoinPoint类型**的参数。这个参数封装了被增强方法的相关信息。**我们可以通过这个参数获取到除了异常对象和返回值之外的所有信息。**

例如：

~~~~java
    @Before("pt()")
    public void methodbefore(JoinPoint jp){
        Object[] args = jp.getArgs();//方法调用时传入的参数
        Object target = jp.getTarget();//被代理对象
        MethodSignature signature = (MethodSignature) jp.getSignature();//获取被被增强方法签名封装的对象
        System.out.println("Before方法被调用了");
    }
~~~~



案例：

需求：要求让所有service包下类的所有方法被调用前都输出全类名，方法名，以及调用时传入的参数

~~~~java
@Component
@Aspect
public class PrintLogAspect {

    //对哪些方法增强
    @Pointcut("execution(* com.sangeng.service..*.*(..))")
    public void pt(){}

    //怎么增强
    @Before("pt()")
    public void printLog(JoinPoint joinPoint){
        //输出 被增强的方法所在的类名 方法名 调用时传入的参数   joinPoint.getSignature().getName()  joinPoint.getArgs()
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        //类名
        String className = signature.getDeclaringTypeName();
        //方法名
        String methodName = signature.getName();
        //调用时传入的参数
        Object[] args = joinPoint.getArgs();

        System.out.println(className+"=="+methodName+"======"+ Arrays.toString(args));
    }
}

~~~~







​	如果需要**获取被增强方法中的异常对象或者返回值**则需要在方法参数上增加一个对应类型的参数，并且使用注解的属性进行配置。这样Spring会把你想获取的数据赋值给对应的方法参数。

例如：

~~~~java
    @AfterReturning(value = "pt()",returning = "ret")//使用returning属性指定了把目标方法返回值赋值给下面方法的参数ret
    public void AfterReturning(JoinPoint jp,Object ret){
        System.out.println("AfterReturning方法被调用了");
    }
~~~~

~~~~java
    @AfterThrowing(value = "pt()",throwing = "t")//使用throwing属性指定了把出现的异常对象赋值给下面方法的参数t
    public void AfterThrowing(JoinPoint jp,Throwable t){
        System.out.println("AfterReturning方法被调用了");
    }
~~~~



​	相信你肯定觉得上面的获取方式特别的麻烦难以理解。就可以使用下面这种万能的方法。

​	直接在环绕通知方法中增加一个**ProceedingJoinPoint类型**的参数。这个参数封装了被增强方法的相关信息。

该参数的proceed()方法被调用相当于被增强方法被执行，调用后的返回值就相当于被增强方法的返回值。

例如：

~~~~java
    @Around(value = "pt()")
    public Object around(ProceedingJoinPoint pjp) {
        Object[] args = pjp.getArgs();//方法调用时传入的参数
        Object target = pjp.getTarget();//被代理对象
        MethodSignature signature = (MethodSignature) pjp.getSignature();//获取被被增强方法签名封装的对象
        Object ret = null;
        try {
            ret = pjp.proceed();//ret就是目标方法执行后的返回值
        } catch (Throwable throwable) {
            throwable.printStackTrace();//throwable就是出现异常时的异常对象
        }
        return ret;
    }
~~~~

> @Around增强方法的返回值，就是调用目标方法的真实返回值。

## xml配置AOP

### ①定义切面类

~~~~java
public class MyAspect {


    public void before(JoinPoint joinPoint){
        System.out.println("before");
    }

//    @AfterReturning(value = "pt()",returning = "ret")
    public void afterReturning(JoinPoint joinPoint,Object ret){
        System.out.println("afterReturning:"+ret);
    }
//    @After("pt()")
    public void after(JoinPoint joinPoint){
        System.out.println("after");
    }

//    @AfterThrowing(value = "pt()",throwing = "e")
    public void afterThrowing(JoinPoint joinPoint,Throwable e){
        String message = e.getMessage();
        System.out.println("afterThrowing:"+message);
    }

    public Object around(ProceedingJoinPoint pjp){
        //获取参数
        Object[] args = pjp.getArgs();
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Object target = pjp.getTarget();
        Object ret = null;
        try {
            ret = pjp.proceed();//目标方法的执行
            //ret就是被增强方法的返回值
            System.out.println(ret);
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            System.out.println(throwable.getMessage());
        }
//        System.out.println(pjp);
        return ret;
    }
}

~~~~



### ②目标类和切面类注入容器

在切面类和目标类上加是对应的注解。注入如果是使用注解的方式注入容器要记得开启组件扫描。

当然你也可以在xml中使用bean标签的方式注入容器。

~~~~java
@Component//把切面类注入容器
public class MyAspect {
	//..。省略无关代码
}
~~~~

~~~~java
@Service//把目标类注入容器
public class UserService {
	//..。省略无关代码
}
~~~~

### ③配置AOP

~~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!--开启组件扫描-->
    <context:component-scan base-package="com.sangeng"></context:component-scan>

    <!--配置AOP-->
    <aop:config>
        <!--定义切点-->
        <aop:pointcut id="pt1" expression="execution(* com.sangeng.service..*.*(..))"></aop:pointcut>
        <aop:pointcut id="pt2" expression="@annotation(com.sangeng.aspect.InvokeLog)"></aop:pointcut>
        <!--配置切面-->
        <aop:aspect ref="myAspect">
            <aop:before method="before" pointcut-ref="pt1"></aop:before>
            <aop:after method="after" pointcut-ref="pt1"></aop:after>
            <aop:after-returning method="afterReturning" pointcut-ref="pt1" returning="ret"></aop:after-returning>
            <aop:after-throwing method="afterThrowing" pointcut-ref="pt2" throwing="e"></aop:after-throwing>
        </aop:aspect>
    </aop:config>
</beans>
~~~~

## 多切面顺序问题

​	在实际项目中我们可能会存在配置了多个切面的情况。这种情况下我们很可能需要控制切面的顺序。

​	我们在默认情况下Spring有它自己的排序规则。（按照类名排序）

​	默认排序规则往往不符合我们的要求，我们需要进行特殊控制。

​	如果是注解方式配置的AOP可以在切面类上加**@Order注解**来控制顺序。**@Order中的属性越小优先级越高。**

​	如果是XML方式配置的AOP,可以通过调整**配置顺序**来控制。

例如：

下面这种配置方式就会先使用CryptAspect里面的增强，在使用APrintLogAspect里的增强

~~~~java
@Component
@Aspect
@Order(2)
public class APrintLogAspect {
    //省略无关代码
}
@Component
@Aspect
@Order(1)
public class CryptAspect {
    //省略无关代码
}
~~~~

## AOP原理-动态代理

​	实际上Spring的AOP其实底层就是使用动态代理来完成的。并且使用了两种动态代理分别是JDK的动态代理和Cglib动态代理。

​	所以我们接下去来学习下这两种动态代理，理解下它们的不同点。



###  JDK动态代理

​	JDK的动态代理使用的java.lang.reflect.Proxy这个类来进行实现的。要求被代理（被增强）的类需要实现了接口。并且JDK动态代理也只能对接口中的方法进行增强。

~~~~java
public static void main(String[] args) {
        AIControllerImpl aiController = new AIControllerImpl();
        //使用动态代理增强getAnswer方法
        //1.JDK动态代理
        //获取类加载器
        ClassLoader cl = Demo.class.getClassLoader();
        //被代理类所实现接口的字节码对象数组
        Class<?>[] interfaces = AIControllerImpl.class.getInterfaces();
        AIController proxy = (AIController) Proxy.newProxyInstance(cl, interfaces, new InvocationHandler() {
            //使用代理对象的方法时 会调用到invoke
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                //proxy   是代理对象
                //method 是当前被调用的方法封装的Method对象
                //args   是调用方法时传入的参数
                //调用被代理对象的对应方法
                //判断 当前调用的是否是getAnswer方法
                if(method.getName().equals("getAnswer")){
                    System.out.println("增强");
                }
                Object ret = method.invoke(aiController, args);
                return ret;
            }
        });
        String answer = proxy.getAnswer("三连了吗？");
		System.out.println(answer);
    }
~~~~



###  Cglib动态代理

​	使用的是org.springframework.cglib.proxy.Enhancer类进行实现的。

~~~~java
public class CglibDemo {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        //设置父类的字节码对象
        enhancer.setSuperclass(AIControllerImpl.class);
        enhancer.setCallback(new MethodInterceptor() {
            //使用代理对象执行方法是都会调用到intercept方法
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                //判断当前调用的方法是不是getAnswer方法 如果是进行增强
                if ("getAnswer".equals(method.getName())){
                    System.out.println("被增强了");
                }
                //调用父类中对应的方法
                Object ret = methodProxy.invokeSuper(o, objects);
                return ret;
            }
        });
        //生成代理对象
        AIControllerImpl proxy = (AIControllerImpl) enhancer.create();
//        System.out.println(proxy.getAnswer("你好吗？"));
        System.out.println(proxy.fortuneTelling("你好吗？"));
    }
}

~~~~



### 总结

​	JDK动态代理要求被代理（被增强）的类必须要实现接口，生成的代理对象相当于是被代理对象的兄弟。

​	Cglib的动态代理不要求被代理（被增强）的类要实现接口，生成的代理对象相当于被代理对象的子类对象。

​	**Spring的AOP默认情况下优先使用的是JDK的动态代理，如果使用不了JDK的动态代理才会使用Cglib的动态代理。**



### 切换默认动态代理方式

​	有的时候我们需要修改AOP的代理方式。

​	我们可以使用以下方式修改：

如果我们是采用注解方式配置AOP的话：

设置aop:aspectj-autoproxy标签的proxy-target-class属性为true，代理方式就会修改成Cglib

~~~~xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
~~~~



如果我们是采用xml方式配置AOP的话：

设置aop:config标签的proxy-target-class属性为true,代理方式就会修改成Cglib

~~~~xml
<aop:config proxy-target-class="true">
</aop:config>
~~~~

# 4.声明式事务

## 事务的概念

保证一组数据库的操作，要么同时成功，要么同时失败

**四大特性**

- 隔离性

  多个事务之间要相互隔离，不能互相干扰

- 原子性

  指事务是一个不可分割的整体，类似一个不可分割的原子

- 一致性

  保障事务前后这组数据的状态是一致的。要么都是成功的，要么都是失败的。

- 持久性

  指事务一旦被提交，这组操作修改的数据就真的的发生变化了。即使接下来数据库故障也不应该对其有影响。

  

## 实现声明式事务

​	如果我们自己去对事务进行控制的话我们就需要值原来核心代码的基础上加上事务控制相关的代码。而在我们的实际开发中这种事务控制的操作也是非常常见的。所以Spring提供了声明式事务的方式让我们去控制事务。

​	**只要简单的加个注解(或者是xml配置)就可以实现事务控制，不需要事务控制的时候只需要去掉相应的注解即可。**

### 注解实现

### ①配置事务管理器和事务注解驱动

在spring的配置文件中添加如下配置：

~~~~xml
    <!--把事务管理器注入Spring容器，需要配置一个连接池-->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    <!--开启事务注解驱动，配置使用的事务管理器-->
    <tx:annotation-driven transaction-manager="txManager"/>
~~~~

### ②添加注解

在需要进行事务控制的方法或者类上添加@Transactional注解就可以实现事务控制。

~~~~java
    @Transactional
    public void transfer(Integer outId, Integer inId, Double money) {
        //增加
        accoutDao.updateMoney(inId,money);
//        System.out.println(1/0);
        //减少
        accoutDao.updateMoney(outId,-money);
    }
~~~~

**注意：如果加在类上，这个类的所有方法都会受事务控制，如果加在方法上，就是那一个方法受事务控制。**

注意，因为声明式事务底层是通过AOP实现的，所以最好把AOP相关依赖都加上。

~~~~xml
       <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.9.6</version>
        </dependency>
~~~~



## xml方式实现

### ①配置事务管理器

~~~~xml
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
~~~~

### ②配置事务切面

~~~~xml
 	<!--定义事务管理的通知类-->
    <tx:advice transaction-manager="txManager" id="txAdvice">
        <tx:attributes>
            <tx:method name="trans*"/>
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="pt" expression="execution(* com.sangeng.service..*.*(..))"></aop:pointcut>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="pt"></aop:advisor>
    </aop:config>
~~~~

注意，因为声明式事务底层是通过AOP实现的，所以最好把AOP相关依赖都加上。

~~~~xml
       <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.9.6</version>
        </dependency>
~~~~

## 属性配置

### 事务传播行为propagation

​	当事务方法嵌套调用时，需要控制是否开启新事务，可以使用事务传播行为来控制。



测试案例:

~~~~java
@Service
public class TestServiceImpl {
    @Autowired
    AccountService accountService;

    @Transactional
    public void test(){
        accountService.transfer(1,2,10D);
        accountService.log();
    }
}
~~~~

~~~~java
public class AccountServiceImpl implements AccountService {
	//...省略其他不相关代码
    @Transactional
    public void log() {
        System.out.println("打印日志");
        int i = 1/0;
    }

}
~~~~



| 属性值                         | 行为                                                   |
| ------------------------------ | ------------------------------------------------------ |
| REQUIRED（必须要有）           | 外层方法有事务，内层方法就加入。外层没有，内层就新建   |
| REQUIRES_NEW（必须要有新事务） | 外层方法有事务，内层方法新建。外层没有，内层也新建     |
| SUPPORTS（支持有）             | 外层方法有事务，内层方法就加入。外层没有，内层就也没有 |
| NOT_SUPPORTED（支持没有）      | 外层方法有事务，内层方法没有。外层没有，内层也没有     |
| MANDATORY（强制要求外层有）    | 外层方法有事务，内层方法加入。外层没有。内层就报错     |
| NEVER(绝不允许有)              | 外层方法有事务，内层方法就报错。外层没有。内层就也没有 |

例如：

~~~~java
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void transfer(Integer outId, Integer inId, Double money) {
        //增加
        accoutDao.updateMoney(inId,money);
        //减少
        accoutDao.updateMoney(outId,-money);
    }
~~~~

### 隔离级别isolation

Isolation.DEFAULT 使用数据库默认隔离级别

Isolation.READ_UNCOMMITTED 

Isolation.READ_COMMITTED

Isolation.REPEATABLE_READ

Isolation.SERIALIZABLE

~~~~java
   @Transactional(propagation = Propagation.REQUIRES_NEW,isolation = Isolation.READ_COMMITTED)
    public void transfer(Integer outId, Integer inId, Double money) {
        //增加
        accoutDao.updateMoney(inId,money);
        //减少
        accoutDao.updateMoney(outId,-money);
    }
~~~~

### 只读readOnly

​	如果事务中的操作都是读操作，没涉及到对数据的写操作可以设置readOnly为true。这样可以提高效率。

~~~~java
    @Transactional(readOnly = true)
    public void log() {
        System.out.println("打印日志");
        int i = 1/0;
    }
~~~~
