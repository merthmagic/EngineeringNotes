# spring mvc笔记 #


## 基础 ##

### Maven ###
一般使用`maven`来管理包依赖。`eclipse`带了内建的`maven`，要在`eclipse`中配置使配置文件指向自己本地安装的`maven`，`maven`需要配置一下镜像和本地仓库。然后可以在`pom`文件中直接引入项目所需的依赖.

普通的小应用创建一个工程即可，如果是大的项目需要建立像Visual Studio中solution那样的多项目结构，需要先建立一个`maven`项目，然后再添加module.

### 引入spring mvc ###

`SpringMVC`类似于.NET平台的`ASP.NET MVC`,但也有很大区别,`spring`组件的核心还是做DI Container.在web app中配置了一个大`servlet`来处理所有的请求，路由系统没有ASP.NET MVC强大，配置也更复杂，但理解深入后，应该比`ASP.NET MVC`条理更清晰.

为了保证`spring`系列的版本统一，`pom`文件中设置一个属性（理解上可以参照nant配置中的属性，也可以认为是一种环境变量).

```xml
<properties>
	<spring.version>4.0.1.RELEASE</spring.version>
</properties>
```

然后在`dependencies`节下面引入`spring mvc`的包
```xml
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-core</artifactId>
	<version>${spring.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-web</artifactId>
	<version>${spring.version}</version>
</dependency>

<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-webmvc</artifactId>
	<version>${spring.version}</version>
</dependency>
```
### 引入servlet包 ###

`eclipse`有抱怨称找不到servlet定义,据@Alvin称这个在部署后不影响，不过比较代码洁癖的话还是不能容忍这个error,引入`servlet-api`包即可解.

```xml
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>servlet-api</artifactId>
	<version>2.5</version>
</dependency>
```

### 配置web.xml ###

注意`web-app`的**version**问题，至少**2.5**，否则`EL表达式`不工作.

直接附一个配置的example
```xml
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
id="Your_WebApp_ID" version="2.5">

	<display-name>mdsd webapp</display-name>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
	</context-param>

	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<servlet>
		<servlet-name>dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>dispatcher</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

	<jsp-config>
		<jsp-property-group>
			<url-pattern>*.jsp</url-pattern>
			<trim-directive-whitespaces>true</trim-directive-whitespaces>
		</jsp-property-group>
	</jsp-config>
	
	<welcome-file-list>
		<welcome-file>index.jsp</welcome-file>
	</welcome-file-list>
</web-app>
```

上面这个例子中，注意`jsp-config`，这是为了消除`@page指令`造成的空白行.

### servlet配置 ###

配置完web.xml还不够，要告诉web容器如何去找controller，如何dispatch.

用一个xml文件来完成这个工作，名字路径要和web.xml中`context-param`配置的一致.

例如就叫dispatcher-servlet.xml，配置代码如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd 
	http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd 
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">
 
	<context:component-scan base-package="com.mdsd.webapp.controllers" />
 
	<bean
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix">
			<value>/WEB-INF/views/</value>
		</property>
		<property name="suffix">
			<value>.jsp</value>
		</property>
	</bean>
</beans>
```

### controller ###

controller概念好理解，直接上代码

```java
package com.mdsd.webapp.controllers;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class HomeController {

	@RequestMapping("/home")
	public ModelAndView index(){
		ModelAndView mv = new ModelAndView("index");
		mv.addObject("name", "hello web");
		mv.addObject("title","A springMVC webapp");
		return mv;
	}
	
	@RequestMapping("/home/greeting")
	public ModelAndView greeting(@RequestParam(value="name",required=false,defaultValue="world")String name){
		ModelAndView mv = new ModelAndView("home/index");
		mv.addObject("name", name);
		return mv;
	}
}
```
要记住的就那几个注解，`@Controller`  `@RequestMapping`  `@RequestParam`.

### mybatis ###

应用必须和数据库交互，为了配置自由度更高，选择`mybatis`

要引入的依赖包括`mysql`数据库的连接库，`mybatis`库，`mybatis`和`spring`交互的库.为了配置数据连接池，还要再引入一个`dbcp`.

```xml
<!-- mybatis核心包 -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis</artifactId>
			<version>${mybatis.version}</version>
		</dependency>
		<!-- mybatis/spring包 -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis-spring</artifactId>
			<version>1.2.2</version>
		</dependency>

		<!-- 导入Mysql数据库链接jar包 -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.30</version>
		</dependency>
		
		<!-- 导入dbcp的jar包，用来在applicationContext.xml中配置数据库 -->
		<dependency>
			<groupId>commons-dbcp</groupId>
			<artifactId>commons-dbcp</artifactId>
			<version>1.2.2</version>
		</dependency>
```
完成这些依赖的引入后，要配置mybatis
首先，要对数据库连接进行配置.创建一个jdbc.properties文件来存放连接信息.

```
driver=com.mysql.jdbc.Driver
url=jdbc:mysql://localhost:3306/mdsd
username=root
password=root
initialSize=0
maxActive=20
maxIdle=20
minIdle=1
maxWait=60000
```

创建配置文件spring-mybatis.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans  
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd  
                        http://www.springframework.org/schema/context  
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd  
                        http://www.springframework.org/schema/mvc  
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd">
	<!-- 自动扫描 -->
	<context:component-scan base-package="com.mdsd.webapp.service" />

	<!-- 引入配置文件 -->
	<bean id="propertyConfigurer"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
		<property name="location" value="classpath:jdbc.properties" />
	</bean>

	<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close">
		<property name="driverClassName" value="${driver}" />
		<property name="url" value="${url}" />
		<property name="username" value="${username}" />
		<property name="password" value="${password}" />
		<!-- 初始化连接大小 -->
		<property name="initialSize" value="${initialSize}"></property>
		<!-- 连接池最大数量 -->
		<property name="maxActive" value="${maxActive}"></property>
		<!-- 连接池最大空闲 -->
		<property name="maxIdle" value="${maxIdle}"></property>
		<!-- 连接池最小空闲 -->
		<property name="minIdle" value="${minIdle}"></property>
		<!-- 获取连接最大等待时间 -->
		<property name="maxWait" value="${maxWait}"></property>
	</bean>

	<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<!-- 自动扫描mapping.xml文件 -->
		<property name="mapperLocations" value="classpath:com/mdsd/webapp/mapping/*.xml"></property>
	</bean>

	<!-- DAO接口所在包名，Spring会自动查找其下的类 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="com.mdsd.webapp.dao" />
		<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
	</bean>

	<!-- (事务管理)transaction manager, use JtaTransactionManager for global tx -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource" />
	</bean>

</beans>
```
这个配置文件包含的东西比较多，核心内容是关于数据源，还有dao等等的获取位置.
配置完这个，要把它包含到web.xml中去，否则不生效.

然后要关心的是sql映射，手写不现实，用工具生成，生成完放到代码文件夹中去，编写测试.













