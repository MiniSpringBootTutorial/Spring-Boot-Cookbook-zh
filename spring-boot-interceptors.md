# Spring Boot：定制拦截器

Servlet 过滤器属于Servlet API，和Spring关系不大。除了使用过滤器包装web请求，Spring MVC还提供*HandlerInterceptor（拦截器）*工具。根据文档，HandlerInterceptor的功能跟过滤器类似，但拦截器提供更精细的控制能力：在request被响应之前、request被响应之后、视图渲染之前以及request全部结束之后。我们不能通过拦截器修改request内容，但是可以通过抛出异常（或者返回false）来暂停request的执行。

Spring MVC中常用的拦截器有：*LocaleChangeInterceptor（用于国际化配置）*和*ThemeChangeInterceptor*。我们也可以增加自己定义的拦截器，可以参考这篇文章中提供的[demo](http://lihao312.iteye.com/blog/2078139)

## How Do

添加拦截器不仅是在WebConfiguration中定义bean，Spring Boot提供了基础类WebMvcConfigurerAdapter，我们项目中的WebConfiguration类需要继承这个类。

1. 继承WebMvcConfigurerAdapter；
2. 为LocaleChangeInterceptor添加@Bean定义，这仅仅是定义了一个interceptor spring bean，但是Spring boot不会自动将它加入到调用链中。
3. 拦截器需要手动加入调用链。

修改后完整的WebConfiguration代码如下：

```
package com.test.bookpub;

import org.apache.catalina.filters.RemoteIpFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;

@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {
    @Bean    public RemoteIpFilter remoteIpFilter() {
        return new RemoteIpFilter();
    }

    @Bean    public LocaleChangeInterceptor localeChangeInterceptor() {
        return new LocaleChangeInterceptor();
    }
    @Override    public void addInterceptors(InterceptorRegistry registry {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

使用`mvn spring-boot:run`运行程序，然后通过httpie访问`http://localhost:8080/books?locale=foo`，在终端看到如下错误信息。

```
Servlet.service() for servlet [dispatcherServlet] in context with path [] 
threw exception [Request processing failed; nested exception is 
java.lang.UnsupportedOperationException: Cannot change HTTP accept 
header - use a different locale resolution strategy] with root cause
```

PS：这里发生错误并不是因为我们输入的locale是错误的，而是因为默认的locale修改策略不允许来自浏览器的请求修改。发生这样的错误说明我们之前定义的拦截器起作用了。

## 分析

在我们的示例项目中，覆盖并重写了*addInterceptors(InterceptorRegistory registory)*方法，这是典型的回调函数——利用该函数的参数registry来添加自定义的拦截器。

在Spring Boot的自动配置阶段，Spring Boot会扫描所有WebMvcConfigurer的实例，并顺序调用其中的回调函数，这表示：如果我们想对配置信息做逻辑上的隔离，可以在Spring Boot项目中定义多个WebMvcConfigurer的实例。
