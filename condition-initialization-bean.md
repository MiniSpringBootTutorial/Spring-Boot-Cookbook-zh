在[Spring Boot：定制自己的starter](http://www.jianshu.com/p/85460c1d835a)一文提到，只要DbCountRunner这个类在classpath路径下，Spring Boot会自动创建对应的spring bean并添加到应用程序上下文中。

在文章最后提到，Spring Boot的自动配置机制依靠*@ConditionalOnMissingBean*注解判断是否执行初始化代码，即如果用户已经创建了bean，则相关的初始化代码不再执行。

现在在上篇文章的基础上进行演示，看看*@ConditionalOnMissingBean*注解的作用。

## How Do
- 在pom文件中增加依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
```
- 在*DbCountAutoConfiguration*类中添加*@ConditionalOnMissingBean*注解，如下所示：
```
@Configuration
public class DbCountAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public DbCountRunner dbCountRunner(Collection<CrudRepository> repositories) {
        return new DbCountRunner(repositories);
    }
}
```
- 启动应用程序后，看到跟上篇文章相同的结果；
- 修改日志级别为DEBUG，可以看到DbCountAutoConfiguration属于*Positive match*组。
![DbCountAutoConfiguration的自动配置信息](http://upload-images.jianshu.io/upload_images/44770-98ec7639fadab074.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 在BookPubApplication类中定义DbCountRunner的spring bean
```
@Bean
public DbCountRunner dbCountRunner(Collection<CrudRepository> repositories) {
    return new DbCountRunner(repositories) {
        @Override
        public void run(String... strings) throws Exception {
            logger.info("Manually Declared DbCountRunner");
        }
    };
}
```
- 再次运行程序，观察结果，可以看到这个配置信息放在Negative matchs组中，显示判断条件不匹配，因为已经找到dbCountRunner这个bean。
![手动配置的Bean优先](http://upload-images.jianshu.io/upload_images/44770-ba478f95effa1853.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![修改后的日志信息，显示手动配置bean](http://upload-images.jianshu.io/upload_images/44770-d5b4ad37f7876dbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
