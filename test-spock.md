Spock框架是基于Groovy语言的测试框架，Groovy与Java具备良好的互操作性，因此可以在Spring Boot项目中使用该框架写优雅、高效以及DSL化的测试用例。Spock通过*@RunWith*注解与JUnit框架协同使用，另外，Spock也可以和Mockito([Spring Boot应用的测试——Mockito](http://www.jianshu.com/p/972cd6b93206))协同使用。

在这个小节中我们会利用Spock、Mockito一起编写一些测试用例（包括对Controller的测试和对Repository的测试），感受下Spock的使用。

## How Do
- 根据[Building an Application with Spring Boot](https://spring.io/guides/gs/spring-boot/)这篇文章的描述，*spring-boot-maven-plugin*这个插件同时也支持在Spring Boot框架中使用Groovy语言。
- 在pom文件中添加Spock框架的依赖

```
<!-- test -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-test</artifactId>
   <scope>test</scope>
</dependency>
<dependency>
   <groupId>org.spockframework</groupId>
   <artifactId>spock-core</artifactId>
   <scope>test</scope></dependency>
<dependency>
   <groupId>org.spockframework</groupId>
   <artifactId>spock-spring</artifactId>
   <scope>test</scope>
</dependency>
```
- 在src/test目录下创建groovy文件夹，在groovy文件夹下创建com/test/bookpub包。
- 在*com/test/bookpub*目录下创建*SpockBookRepositorySpecification.groovy*文件，内容是：

```
@WebAppConfiguration
@ContextConfiguration(classes = [BookPubApplication.class,
 TestMockBeansConfig.class],loader = SpringApplicationContextLoader.class)
class SpockBookRepositorySpecification extends Specification {
    @Autowired
    private ConfigurableApplicationContext context;
    @Shared
    boolean sharedSetupDone = false;
    @Autowired
    private DataSource ds;
    @Autowired
    private BookRepository bookRepository;
    @Autowired
    private PublisherRepository publisherRepository;
    @Shared
    private MockMvc mockMvc;

    void setup() {
        if (!sharedSetupDone) {
            mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
            sharedSetupDone = true;
        }
        ResourceDatabasePopulator populator = new 
               ResourceDatabasePopulator(context.getResource("classpath:/packt-books.sql"));
        DatabasePopulatorUtils.execute(populator, ds);
    }

    @Transactional
    def "Test RESTful GET"() {
        when:
        def result = mockMvc.perform(get("/books/${isbn}"));
  
        then:
        result.andExpect(status().isOk()) 
       result.andExpect(content().string(containsString("title")));

       where:
       isbn              | title
      "978-1-78398-478-7"|"Orchestrating Docker"
      "978-1-78528-415-1"|"Spring Boot Recipes"
    }

    @Transactional
    def "Insert another book"() {
      setup:
      def existingBook = bookRepository.findBookByIsbn("978-1-78528-415-1")
      def newBook = new Book("978-1-12345-678-9", "Some Future Book",
              existingBook.getAuthor(), existingBook.getPublisher())

      expect:
      bookRepository.count() == 3

      when:
      def savedBook = bookRepository.save(newBook)

      then:
      bookRepository.count() == 4
      savedBook.id > -1
  }
}
```
- 执行测试用例，测试通过
- 接下来试验下Spock如何与mock对象一起工作，之前的文章中我们已经在*TestMockBeansConfig*类中定义了*PublisherRepository*的Spring Bean，如下所示，由于@Primary的存在，使得在运行测试用例时Spring Boot优先使用Mockito框架模拟出的实例。

```
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
- 在BookController.java中添加getBooksByPublisher接口，代码如下所示：

```
@Autowired
public PublisherRepository publisherRepository;

@RequestMapping(value = "/publisher/{id}", method = RequestMethod.GET)
public List<Book> getBooksByPublisher(@PathVariable("id") Long id) {
    Publisher publisher = publisherRepository.findOne(id);
    Assert.notNull(publisher);
    return publisher.getBooks();
}
```
- 在*SpockBookRepositorySpecification.groovy*文件中添加对应的测试用例，

```
def "Test RESTful GET books by publisher"() {
    setup:
    Publisher publisher = new Publisher("Strange Books")
    publisher.setId(999)
    Book book = new Book("978-1-98765-432-1",
            "Mytery Book",
            new Author("Jhon", "Done"),
            publisher)
    publisher.setBooks([book])
    Mockito.when(publisherRepository.count()).
            thenReturn(1L);
    Mockito.when(publisherRepository.findOne(1L)).
            thenReturn(publisher)

    when:
    def result = mockMvc.perform(get("/books/publisher/1"))

    then:
    result.andExpect(status().isOk())
    result.andExpect(content().string(containsString("Strange Books")))

    cleanup:
    Mockito.reset(publisherRepository)
}
```
- 运行测试用例，发现可以测试通过，在控制器将对象转换成JSON字符串装入HTTP响应体时，依赖Jackson库执行转换，可能会有循环依赖的问题——在模型关系中，一本书依赖一个出版社，一个出版社有包含多本书，在执行转换时，如果不进行特殊处理，就会循环解析。我们这里通过*@JsonBackReference*注解阻止循环依赖。

## 分析
可以看出，通过Spock框架可以写出优雅而强大的测试代码。

首先看SpockBookRepositorySpecification.groovy文件，该类继承自Specification类，告诉JUnit这个类是测试类。查看Specification类的源码，可以发现它被@RunWith(Sputnik.class)注解修饰，这个注解是连接Spock与JUnit的桥梁。除了引导JUnit，Specification类还提供了很多测试方法和mocking支持。

>Note：关于Spock的文档见这里：[Spock Framework Reference Documentation](http://spockframework.github.io/spock/docs/1.0/index.html)

根据《单元测试的艺术》一书中提到的，单元测试包括：准备测试数据、执行待测试方法、判断执行结果三个步骤。Spock通过setup、expect、when和then等标签将这些步骤放在一个测试用例中。
- setup：这个块用于定义变量、准备测试数据、构建mock对象等；
- expect：一般跟在setup块后使用，包含一些assert语句，检查在setup块中准备好的测试环境
- when：在这个块中调用要测试的方法；
- then : 一般跟在when后使用，尽可以包含断言语句、异常检查语句等等，用于检查要测试的方法执行后结果是否符合预期；
- cleanup：用于清除setup块中对环境做的修改，即将当前测试用例中的修改回滚，在这个例子中我们对publisherRepository对象执行重置操作。

Spock也提供了setup()和cleanup()方法，执行一些给所有测试用例使用的准备和清除动作，例如在这个例子中我们使用setup方法：（1）mock出web运行环境，可以接受http请求；（2）加载packt-books.sql文件，导入预定义的测试数据。web环境只需要Mock一次，因此使用sharedSetupDone这个标志来控制。

通过@Transactional注解可以实现事务操作，如果某个方法被该注解修饰，则与之相关的setup()方法、cleanup()方法都被定义在一个事务内执行操作：要么全部成功、要么回滚到初始状态。我们依靠这个方法保证数据库的整洁，也避免了每次输入相同的数据。
