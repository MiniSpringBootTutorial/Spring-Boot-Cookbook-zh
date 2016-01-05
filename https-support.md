如今，企业级应用程序的常见场景是同时支持HTTP和HTTPS两种协议，这篇文章考虑如何让Spring Boot应用程序同时支持HTTP和HTTPS两种协议。

## 准备
为了使用HTTPS连接器，需要生成一份Certificate keystore，用于加密和机密浏览器的SSL沟通。

如果你使用Unix或者Mac OS，可以通过下列命令：`keytool -genkey -alias tomcat -keyalg RSA`，在生成过程中可能需要你填入一些自己的信息，例如我的机器上反馈如下：

![生成kestore](http://upload-images.jianshu.io/upload_images/44770-8cedf2748dac62be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出，执行完上述命令后在home目录下多了一个新的.keystore文件。

## How Do
- 首先在resources目录下新建一个配置文件*tomcat.https.properties*，用于存放HTTPS的配置信息；

```
custom.tomcat.https.port=8443
custom.tomcat.https.secure=true
custom.tomcat.https.scheme=https
custom.tomcat.https.ssl=true
custom.tomcat.https.keystore=${user.home}/.keystore
custom.tomcat.https.keystore-password=changeit
```
- 然后在WebConfiguration类中创建一个静态类*TomcatSslConnectorProperties*；

```
@ConfigurationProperties(prefix = "custom.tomcat.https")
public static class TomcatSslConnectorProperties {
    private Integer port;
    private Boolean ssl = true;
    private Boolean secure = true;
    private String scheme = "https";
    private File keystore;
    private String keystorePassword;
    //这里为了节省空间，省略了getters和setters，读者在实践的时候要加上
    
    public void configureConnector(Connector connector) {
        if (port != null) {
            connector.setPort(port);
        }
        if (secure != null) {
            connector.setSecure(secure);
        }
        if (scheme != null) {
            connector.setScheme(scheme);
        }
        if (ssl != null) {
            connector.setProperty("SSLEnabled", ssl.toString());
        }
        if (keystore != null && keystore.exists()) {
            connector.setProperty("keystoreFile", keystore.getAbsolutePath());
            connector.setProperty("keystorePassword", keystorePassword);
        }
    }
}
```
- 通过注解加载*tomcat.https.properties*配置文件，并与*TomcatSslConnectorProperties*绑定，用注解修饰WebConfiguration类；

```
@Configuration
@PropertySource("classpath:/tomcat.https.properties")
@EnableConfigurationProperties(WebConfiguration.TomcatSslConnectorProperties.class)
public class WebConfiguration extends WebMvcConfigurerAdapter {...}
```
- 在WebConfiguration类中创建*EmbeddedServletContainerFactory*类型的Srping bean，并用它添加之前创建的HTTPS连接器。

```
@Bean
public EmbeddedServletContainerFactory servletContainer(TomcatSslConnectorProperties properties) {
    TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
    tomcat.addAdditionalTomcatConnectors(createSslConnector(properties));
    return tomcat;
}

private Connector createSslConnector(TomcatSslConnectorProperties properties) {
    Connector connector = new Connector();
    properties.configureConnector(connector);
    return connector;
}
```

- 通过`mvn spring-boot:run`启动应用程序；
- 在浏览器中访问URL`https://localhost:8443/internal/tomcat.https.properties`

![支持HTTPS协议](http://upload-images.jianshu.io/upload_images/44770-7b57b3273d0b35b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 在浏览器中访问URL`http://localhost:8080/internal/application.properties`

![同时支持HTTP协议](http://upload-images.jianshu.io/upload_images/44770-bb2fc4f49363d7e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 分析
根据之前的文章和官方文档，Spring Boot已经对外开放了很多服务器配置，这些配置信息通过Spring Boot内部的*ServerProperties*类完成绑定，若要参考Spring Boot的通用配置项，请[点击这里](http://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)

>Spring Boot不支持通过application.properties同时配置HTTP连接器和HTTPS连接器。在[官方文档70.8](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-embedded-servlet-containers.html)中提到一种方法，是将属性值硬编码在程序中。

因此我们这里新建一个配置文件*tomcat.https.properties*来实现，但是这并不符合“Spring Boot风格”，后续有可能应该会支持“通过application.properties同时配置HTTP连接器和HTTPS连接器”。我添加的*TomcatSslConnectorProperties*是模仿Spring Boot中的*ServerProperties*的使用机制实现的，这里使用了自定义的属性前缀*custom.tomcat*而没有用现有的*server.*前缀，因为ServerProperties禁止在其他的配置文件中使用该命名空间。

*@ConfigurationProperties(prefix = "custom.tomcat.https")*这个注解会让Spring Boot自动将*custom.tomcat.https*开头的属性绑定到TomcatSslConnectorProperties这个类的成员上（确保该类的getters和setters存在）。值得一提的是，在绑定过程中Spring Boot会自动将属性值转换成合适的数据类型，例如*custom.tomcat.https.keystore*的值会自动绑定到File对象keystore上。

使用*@PropertySource("classpath:/tomcat.https.properties")*来让Spring Boot加载*tomcat.https.properties*文件中的属性。

使用*@EnableConfigurationProperties(WebConfiguration.TomcatSslConnectorProperties.class)*让Spring Boot自动创建一个属性对象，包含上述通过@PropertySource导入的属性。

在属性值导入内存，并构建好TomcatSslConnectorProperties实例后，需要创建一个*EmbeddedServletContainerFactory*类型的Spring bean，用于创建EmbeddedServletContainer。

通过*createSslConnector*方法可以构建一个包含了我们指定的属性值的连接器，然后通过*tomcat.addAdditionalTomcatConnectors(createSslConnector(properties));*设置tomcat容器的HTTPS连接器。



## 参考资料
1. [配置SSL](https://qbgbook.gitbooks.io/spring-boot-reference-guide-zh/content/IX.%20%E2%80%98How-to%E2%80%99%20guides/64.5.%20Configure%20SSL.html)
