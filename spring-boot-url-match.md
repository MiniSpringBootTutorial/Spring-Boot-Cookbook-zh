构建web应用程序时，并不是所有的URL请求都遵循默认的规则。有时，我们希望RESTful URL匹配的时候包含定界符“.”，这种情况在Spring中可以称之为“定界符定义的格式”；有时，我们希望识别斜杠的存在。Spring提供了接口供开发人员按照需求定制。

在之前的几篇文章中，可以通过WebConfiguration类来定制程序中的过滤器、格式化工具等等，同样得，也可以在这个类中用类似的办法配置“路径匹配规则”。

假设ISBN格式允许通过定界符“.”分割图书编号和修订号，形如**[isbn-number].[revision]**

## How Do
- 在WebConfiguration类中添加对应的配置，代码如下：

```
@Overridepublic 
void configurePathMatch(PathMatchConfigurer configurer) {
    configurer.setUseSuffixPatternMatch(false).
            setUseTrailingSlashMatch(true);
}
```

- 通过`mvn spring-boot:run`启动应用程序
- 访问`http://localhost:8080/books/9781-1234-1111.1`

![在路径匹配时，不使用后缀模式匹配（.*）](http://upload-images.jianshu.io/upload_images/44770-4c89f73a36f1a479.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 访问`http://localhost:8080/books/9781-1234-1111`

![使用正确的URL访问的结果](http://upload-images.jianshu.io/upload_images/44770-f1d22e9bb2a0f14a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 分析
*configurePathMatch(PathMatchConfigurer configurer)*函数让开发人员可以根据需求定制URL路径的匹配规则。
- configurer.setUseSuffixPatternMatch(false)表示设计人员希望系统对外暴露的URL不会识别和匹配.*后缀。在这个例子中，就意味着Spring会将9781-1234-1111.1当做一个{isbn}参数传给BookController。
- configurer.setUseTrailingSlashMatch(true)表示系统不区分URL的最后一个字符是否是斜杠/。在这个例子中，就意味着`http://localhost:8080/books/9781-1234-1111`和`http://localhost:8080/books/9781-1234-1111/`含义相同。

如果需要定制path匹配发生的过程，可以提供自己定制的*PathMatcher*和*UrlPathHelper*，但是这种需求并不常见。
