# 通过EmbeddedServletContainerCustomizer接口调优Tomcat

通过在application.properties设置对应的key-value对，可以配置Spring Boot应用程序的很多特性，例如POST、SSL、MySQL等等。如果需要更加复杂的调优，则可以利用Spring Boot提供的EmbeddedServletContainerCustomizer接口通过编程方式和修改配置信息。

尽管可以通过application.properties设置server.session-timeout属性来配置服务器的会话超时时间，这里我们用EmbeddedServletContainerCustomizer接口修改，来说明该接口的用法。

## How Do

- 假设我们希望设置会话的超时时间为1分钟。在WebConfiguration类中增加EmbeddedServletContainerCustomizer类型的spring bean，代码如下：

```
@Bean
public EmbeddedServletContainerCustomizer embeddedServletContainerCustomizer() {
    return new EmbeddedServletContainerCustomizer() {
        @Override 
        public void customize(ConfigurableEmbeddedServletContainer container) {
            container.setSessionTimeout(1, TimeUnit.MINUTES);
        }
    };
}
```

- 在BookController中添加一个*getSessionId(HttpServletRequest request)*函数，直接返回request.getSession().getId()。

```
@RequestMapping(value = "/session", method = RequestMethod.GET)
public String getSessionId(HttpServletRequest request) {
    return request.getSession().getId();
}
```

- 通过`mvn spring-boot:run`启动应用
- 通过postman访问`http://localhost:8080/books/session`，得到的结果如下

![获取session](http://upload-images.jianshu.io/upload_images/44770-bf1b8f2930b01355.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1分钟以后再次调用这个接口，则发现返回的session id已经改变。

## 分析

除了可以使用上面这个写法，对于使用Java 8的开发人员，还可以使用lambda表达式处理，就不需要创建一个EmbeddedServletContainerCustomizer实例了。代码如下：

```
//对于Java 8来说可以用lambda表达式,而不需要创建该接口的一个实例.
@Bean
public EmbeddedServletContainerCustomizer embeddedServletContainerCustomizer() {
    return (ConfigurableEmbeddedServletContainer container) -> {
        container.setSessionTimeout(1, TimeUnit.MINUTES);
    };
}
```

在程序启动阶段，Spring Boot检测到custoimer实例的存在，然后就会调用invoke(...)方法，并向内传递一个servlet对象的实例。在我们这个例子中，实际上传入的是*TomcatEmbeddedServletContainerFactory*容器对象，但是如果使用Jutty或者Undertow容器，就会用对应的容器对象。
