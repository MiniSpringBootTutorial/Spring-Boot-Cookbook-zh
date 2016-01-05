Spring Boot工程的默认web容器是Tomcat，但是开发人员可以根据需要修改，例如使用Jetty或者Undertow，Spring Boot提供了对应的starters。

## How Do
- 在pom文件中排除tomcat的starter

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-web</artifactId>
   <exclusions>
      <exclusion>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-tomcat</artifactId>
      </exclusion>
   </exclusions>
</dependency>
```
- 增加Jetty依赖

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
- 通过`mvn spring-boot:run`命令启动，可以看到Jetty已经启动。

![Jetty容器启动](http://upload-images.jianshu.io/upload_images/44770-18e4e74400362556.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

PS：如果您使用的gradle，则可以参考官方文档的做法——[Use Jetty instead of Tomcat](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-embedded-servlet-containers.html)

## 分析
支持上述切换的原因是Spring Boot的自动配置。我首先在*spring-boot-starter-web*依赖中排除Tomcat依赖，免得它跟Jetty形成依赖冲突。Spring Boot根据在classpath下扫描到的容器类的类型决定使用哪个web容器。

在IDEA中查看*EmbeddedServletContainerAutoConfiguration*类的内部结构，可以看到`@ConditionalOnClass({Servlet.class, Server.class, Loader.class, WebAppContext.class})`这样的条件匹配注解，如果在Jetty的Jar包中可以找到上述三个类的实例，则决定使用jetty容器。

```
@Configuration
@ConditionalOnClass({Servlet.class, Server.class, Loader.class, WebAppContext.class})
@ConditionalOnMissingBean(    value = {EmbeddedServletContainerFactory.class},    search = SearchStrategy.CURRENT)
public static class EmbeddedJetty {
    public EmbeddedJetty() {
    }
    @Bean
    public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
        // 返回容器工厂实例，用于构造容器实例
        return new JettyEmbeddedServletContainerFactory();
    }
}
```

同样得，可以看到对Tomcat也存在类似的判断和使用代码：

```
@Configuration
@ConditionalOnClass({Servlet.class, Tomcat.class})
@ConditionalOnMissingBean(    value = {EmbeddedServletContainerFactory.class},    search = SearchStrategy.CURRENT)
public static class EmbeddedTomcat {
    public EmbeddedTomcat() {
    }
    @Bean
    public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
        return new TomcatEmbeddedServletContainerFactory();
    }
}
```
