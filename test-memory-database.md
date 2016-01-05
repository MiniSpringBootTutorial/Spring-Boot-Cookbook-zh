在[初始化数据库和导入数据](http://www.jianshu.com/p/468a8fa752a7)一文中，我们探索了在Spring Boot项目中如何创建数据库的表结构，以及如何往数据库中填充初始数据。在程序开发过程中常常会在环境配置上浪费很多时间，例如在一个存在数据库组件的应用程序中，测试用例运行之前必须保证数据库中的表结构正确，并且已经填入初始数据。对于良好的测试用例，还需要保证数据库在执行用例前后状态不改变。

在之前应用的基础上，*schema.sql*文件中包含创建数据库表结构的SQL语句、*data.sql*文件中包含填充初始数据的SQL语句。这篇文章将//todo

## How Do
- 在src/test/resources目录下创建*test-data.sql*文件，用于导入测试数据

```
INSERT INTO author(first_name, last_name) VALUES ("Greg", "Turnquist");
INSERT INTO book(isbn, title, author, publisher) VALUES ('9781-78439-302-1', 'Learning Spring Boot', 2, 1)
```
- 修改*BookPubApplicationTests*文件，添加数据源属性（ds），是否需要导入测试数据的标志（loadDataFixtures），添加loadDataFixtures方法。

```
public class BookPubApplicationTests {
    ...
    @Autowired
    private DataSource ds;
    private static boolean loadDataFixtures = true;
    private MockMvc mockMvc;
    private RestTemplate restTemplate = new TestRestTemplate();

    @Before
    public void setupMockMvc() {
       ...
    }

    @Before
    public void loadDataFixtures() {
        if (loadDataFixtures) {
            ResourceDatabasePopulator populator = new
                    ResourceDatabasePopulator(context.getResource("classpath:/test-data.sql"));
            DatabasePopulatorUtils.execute(populator, ds);
            loadDataFixtures = false;
        }
    }

   @Test
   public void contextLoads() {
        assertEquals(2, bookRepository.count());
   }

    @Test    public void webappBookIsbnApi() {
        ...
    }
    @Test    public void webappPublisherApi() throws Exception {
        ...
    }
}
```
- 运行单元测试，可以通过
- Spring Boot会搜集resources目录下的所有*data.sql文件进行数据导入，由于测试代码有自己的resource目录，因此在这个目录下再创建一个*data.sql*文件，内容是：

```
INSERT INTO author (id, first_name, last_name) VALUES (3, "William", "Shakespeare");
INSERT INTO publisher (id, name) VALUES (2, "Classical Books");
INSERT INTO book(isbn, title, author, publisher) VALUES ('978-1-23456-789-1', "Romeo and Juliet", 3, 2);
```
- 由于又新加了一个book记录，因此需要修改BookPubApplicationTest

```
@Test
public void contextLoads() {
       assertEquals(3, bookRepository.count());
}
```
- 至此我们还都是使用外部数据库——MySQL，现在尝试使用内存数据库H2，因此在*src/test/resources*目录下添加*application.properties*文件，内容是：

```
spring.datasource.url=\  jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.jpa.hibernate.ddl-auto=none
```
- 执行测试用例，可以通过。需要注意的是：Spring Boot仅仅会加载一个*application.properties*文件，由于此处我在src/test/resources目录下新建了application.properties文件，所以之前的那个（在src/main/resources目录下）不会被加载。

## 分析
我们通过Spring的ResourceDatabasePopulator和DatabasePopulatorUtils类加载test-data.sql文件，在test-data.sql文件中的数据仅仅对当前所在的*Test.java文件有效。Spring Boot自身去处理schema.sql和data.sql文件时也是依靠这两个类，这里我们不过是显式指定了我们希望执行的脚本文件。
- 创建setup方法——*loadDataFixtures()*，并用@Before注解修饰，表示在测试用例之前运行该方法。
- 在*loadDataFixtures()*方法中，首先通过context.getResources方法加载test-data.sql文件，然后通过*DatabasePopulatorUtils.execute(populator, ds)*执行该文件中的SQL语句。
- 为了避免每个@Test修饰的测试用例之前都导入数据，因此引入一个标志变量——loadDataFixtures（初始化为true），因此该方法只执行一次。
