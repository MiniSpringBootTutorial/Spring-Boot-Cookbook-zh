接下来关于SpringBoot的一系列文章和例子，都来自《Spring Boot Cookbook》这本书，本文的主要内容是start.spring.io的使用、Spring Boot的自动配置以及CommandRunner的角色和应用场景。


## 1. start.spring.io的使用
首先带你浏览[http://start.spring.io/](http://start.spring.io)，在这个网址中有一些Spring Boot提供的组件，然后会给你展示如何让你的Spring工程变得“Bootiful”，我称之为“Boot化”。

在网站[Spring Initializr](http://start.spring.io/)上填对应的表单，描述Spring Boot项目的主要信息，例如Project Metadata、Dependency等。在Project Dependencies区域，你可以根据应用程序的功能需要选择相应的starter。

Spring Boot starters可以简化Spring项目的库依赖管理，将某一特定功能所需要的依赖库都整合在一起，就形成一个starter，例如：连接数据库、springmvc、spring测试框架等等。简单来说，spring boot使得你的pom文件从此变得很清爽且易于管理。

常用的starter以及用处可以列举如下：

- spring-boot-starter: 这是核心Spring Boot starter，提供了大部分基础功能，其他starter都依赖于它，因此没有必要显式定义它。
- spring-boot-starter-actuator：主要提供监控、管理和审查应用程序的功能。
- spring-boot-starter-jdbc：该starter提供对JDBC操作的支持，包括连接数据库、操作数据库，以及管理数据库连接等等。
- spring-boot-starter-data-jpa：JPA starter提供使用Java Persistence API(例如Hibernate等)的依赖库。
- spring-boot-starter-data-*：提供对MongoDB、Data-Rest或者Solr的支持。
- spring-boot-starter-security：提供所有Spring-security的依赖库。
- spring-boot-starter-test：这个starter包括了spring-test依赖以及其他测试框架，例如JUnit和Mockito等等。
- spring-boot-starter-web：该starter包括web应用程序的依赖库。

### How do
首先我们要通过start.spring.io创建一个图书目录管理程序，它会记录出版图书的记录，包括作者、审阅人、出版社等等。我们将这个项目命名为BookPub，具体的操作步骤如下：

 - 点击“Switch to the full version.”，展示完整页面；
 - **Group**设置为：org.test；
 - **Artifact**设置为：bookpub；
 - **Name**设置为：BookPub；
 - **Package Name**设置为：org.test.bookpub；
 - **Packaging**代表打包方式，我们选jar；
 - **Spring Boot Version**选择最新的1.3.0；
 - 创建Maven工程，当然，对Gradle比较熟悉的同学可以选择Gradle工程。
 - 点击“Generate Project”下载工程包。
 
利用IDEA导入下载的工程，可以看到pom文件的主体如下如下所示：

```

	<groupId>com.test</groupId>
	<artifactId>bookpub</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
	
	<name>BookPub</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.3.0.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	
	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

```

## 2. Spring Boot的自动配置

在Spring Boot项目中，xxxApplication.java会作为应用程序的入口，负责程序启动以及一些基础性的工作。**@SpringBootApplication**是这个注解是该应用程序入口的标志，然后有熟悉的main函数，通过`SpringApplication.run(xxxApplication.class, args)`来运行Spring Boot应用。打开SpringBootApplication注解可以发现，它是由其他几个类组合而成的：@Configuration（等同于spring中的xml配置文件，使用Java文件做配置可以检查类型安全）、@EnableAutoConfiguration（自动配置，稍后细讲）、@ComponentScan（组件扫描，大家非常熟悉的，可以自动发现和装配一些Bean）。

我们在pom文件里可以看到，com.h2database这个库起作用的范围是runtime，也就是说，当应用程序启动时，如果Spring Boot在classpath下检测到org.h2.Driver的存在，会自动配置H2数据库连接。现在启动应用程序来观察，以验证我们的想法。打开shell，进入项目文件夹，利用`mvn spring-boot:run`启动应用程序，如下图所示。

![Spring Boot的自动配置](http://7sblhh.com1.z0.glb.clouddn.com/SPRING%20BOOT启动界面.png)

可以看到类似*Building JPA container EntityManagerFactory for persistence unit 'default*、*HHH000412: Hibernate Core {4.3.11.Final}*、*HHH000400: Using dialect: org.hibernate.dialect.H2Dialect*这些Info信息；由于我们之前选择了jdbc和jpa等starters,Spring Boot将自动创建JPA容器，并使用Hibernate4.3.11，使用H2Dialect管理H2数据库（内存数据库）。

## 3. 使用Command-line runners
我们新建一个StartupRunner类，该类实现CommandLineRunner接口，这个接口只有一个函数：`public void run(String... args)`，最重要的是：**这个方法会在应用程序启动后首先被调用**。

### How do
- 在src/main/java/org/test/bookpub/下建立StartRunner类，代码如下：

```
package com.test.bookpub;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;

public class StartupRunner implements CommandLineRunner {
    protected final Logger logger = LoggerFactory.getLogger(StartupRunner.class);
    @Override
    public void run(String... strings) throws Exception {
        logger.info("hello");
    }
}
```
- 在BookPubApplication类中创建bean对象，代码如下：

```
    @Bean
    public StartupRunner schedulerRunner() {
        return new StartupRunner();
    }
```

还是用`mvn spring-boot:run`命令启动程序，可以看到hello的输出。对于那种只需要在应用程序启动时执行一次的任务，非常适合利用Command line runners来完成。Spring Boot应用程序在启动后，会遍历CommandLineRunner接口的实例并运行它们的run方法。也可以利用@Order注解（或者实现Order接口）来规定所有CommandLineRunner实例的运行顺序。

利用command-line runner的这个特性，再配合依赖注入，可以在应用程序启动时后首先引入一些依赖bean，例如data source、rpc服务或者其他模块等等，这些对象的初始化可以放在run方法中。不过，需要注意的是，在run方法中执行初始化动作的时候一旦遇到任何异常，都会使得应用程序停止运行，因此最好利用try/catch语句处理可能遇到的异常。
