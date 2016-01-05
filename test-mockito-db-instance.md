![mockito.jpg](http://upload-images.jianshu.io/upload_images/44770-fe16af760e764e59.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
前两篇文章的主要内容是：为了给执行测试，如何建立数据库表和导入初始数据。这里我们将学习如何利用Mockito框架和一些注解模拟（mock）*Repository*实例，从而使得测试用例不依赖外部的数据库服务。

我们需要创建一个Spring Boot配置类，在该类中定义用于测试的Spring Bean；我们通过注解指示Spring Boot何时加载测试配置类以及何时执行该类中的代码。在改配置类中，我们将使用Mockito框架创建一些带预定义方法的mock对象，Spring Boot在执行测试用例之前会将这些对象织入。

## How Do
- 首先创建一个注解用于标识仅用于测试的配置类，可以按照如下方法修改BookPubApplication类。可以看出，关键语句*@ComponentScan(excludeFilters = @ComponentScan.Filter(UsedForTesting.class))*表示：程序正式运行时不扫描*@UsedForTesting*修饰的类。

```
@Configuration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = @ComponentScan.Filter(UsedForTesting.class))
@EnableScheduling
public class BookPubApplication {
    public static void main(String[] args) {
        SpringApplication.run(BookPubApplication.class, args);
    }
    @Bean
    public StartupRunner schedulerRunner() {
        return new StartupRunner();
    }
}

@interface UsedForTesting {}
```
- 在src/test/java/com/test/bookpub目录下创建*TestMockBeansConfig*文件，内容是：

```
package com.test.bookpub;

import com.test.bookpub.repository.PublisherRepository;
import org.mockito.Mockito;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
@UsedForTesting
public class TestMockBeansConfig {
    @Bean
    @Primary
    public PublisherRepository createMockPublisherRepository() {
        return Mockito.mock(PublisherRepository.class);
    }
}
```

- 新建一个测试类——*PublisherRepositoryTests*，主要是因为BookPubApplicationTest中的内容太多太乱了（在实际项目中我们会严格限制每个测试类中的内容）。

```
package com.test.bookpub;
import com.test.bookpub.repository.PublisherRepository;
import org.junit.After;import org.junit.Before;import org.junit.Test;
import org.junit.runner.RunWith;import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.IntegrationTest;
import org.springframework.boot.test.SpringApplicationConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import static org.junit.Assert.assertEquals;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = {
        BookPubApplication.class,
        TestMockBeansConfig.class
})
@IntegrationTest
public class PublisherRepositoryTests {
    @Autowired
    private PublisherRepository repository;

    @Before
    public void setupPublisherRepositoryMock() {
        Mockito.when(repository.count())
                .thenReturn(1L);
    }
    @Test
    public void publishersExist() {
        assertEquals(1, repository.count());
    }

    @After
    public void resetPublisherRepositoryMock() {
        Mockito.reset(repository);
    }
}
```
## 分析
OK，分析下刚刚发生了什么。首先，我们从对BookPubApplication.java的修改开始：
- *@SpringBootApplication*被三个注解替换：*@Configuration*, *@EnableAutoConfiguration*和*@ComponentScan(excludeFilters = @ComponentScan.Filter(UsedForTesting.class))*，这么做的原因是可以给@ComponentScan注解增加*excludeFilters*属性，通过这个属性，我们提示Spring Boot在正式运行时忽略被*@UsedForTesting*修饰的类。
- *@UsedForTesting*注解定义在BookPubApplication.java文件中，用于修饰*TestMockBeansConfig*类。

接下来看看在TestMockBeansConfig中的操作，
- @Configuration注解说明这是一个配置类，该类含有应用程序上下文，如果被其他配置文件引入，则该类中定义的Spring Bean应该加入到已经创建的应用上下文。
- 修饰*createMOckPublisherRepository*方法的注解*@Primary*表示：如果在织入的时候发现有多个PublisherRepository的Spring Bean，则让Spring Boot优先使用该方法返回的Spring Bean。在应用程序启动时，Spring Boot根据*@RepositoryRestResource*注解，已经生成一个PublisherRepository的实例，但是这里我们希望应用程序不使用这个真实的实例，而使用Mockito框架模拟出的PublisherRepository实例。

最后看下我们的测试用例，主要关注*setupPublisherRepositoryMock*方法和*resetPublisherRepositoryMock*方法：
- *setupPublisherRepositoryMock*方法被@Before注解修饰，表示在测试用例运行之前被调用，在这个方法中我们配置了mock对象的行为：如果收到*repository.count()*调用，则返回1。Mockito框架提供了很多DSL形式的语句，可以用于定义这些容易理解的规则。

- *resetPublisherRepositoryMock*方法被@After注解修饰，在测试用例执行过后调用，用于清楚之前对repository的设置。
