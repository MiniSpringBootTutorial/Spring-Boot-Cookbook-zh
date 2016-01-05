在[Spring Boot：定制URL匹配规则](http://www.jianshu.com/p/02bff08fcced)一文中我们展示了如何调整URL请求匹配到对应的控制器方法的规则。类似得，也可以控制应用程序对静态文件（前提是被打包进部署包）的处理。

假设我们需要通过URL`http://localhost:8080/internal/application.properties`对外暴露当前程序的配置。

## How Do
- 在WebConfiguration类中添加相应的配置，代码如下：

```
@Overridepublic 
void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/internal/**").
            addResourceLocations("/classpath:/");
}
```
- 通过`mvn spring-boot:run`启动应用程序
- 通过postman访问`http://localhost:8080/internal/application.properties`就得到下列的结果

![通过配置项对外暴露程序的配置信息](http://upload-images.jianshu.io/upload_images/44770-fab76d2fbd9170e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 分析
通过*addResourceHandlers(ResourceHandlerRegistry registry) *方法可以为应用程序中位于classpath路径下或文件系统下的静态资源配置对应的URL，供其他人通过浏览器访问。在这个例子中，我们规定所有以“/internal”开头的URL请求会在classpath:/目录下查找信息。

- *registry.addResourceHandler("/internal/**")*方法添加一个资源处理器，用于注册程序中的静态资源，该函数返回一个*ResourceHandlerRegistration*对象，这个对象可以进一步配置。/internal/**字符串是一个路径模式串，PathMatcher接口用它匹配对应的URL请求，这里默认使用*AntPathMatcher*进行匹配。
- 由上个方法返回的*ResourceHandlerRegistration*实例调用*addResourceLocations("/classpath:/")*方法来规定从哪个目录下加载资源文件。这个目录路径或者是有效的文件系统路径，或者是classpath路径。

PS：通过*setCachePeriod(Interger cachePeriod)*方法可以设置资源处理器的缓存周期——每隔cachePeriod秒就缓存一次。
