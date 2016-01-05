Spring Boot大大简化了持久化任务，几乎不需要写SQL语句，之前我写过一篇关于Mongodb的——[RESTful:Spring Boot with Mongodb](http://duqicauc.github.io/2015/11/29/RESTful-Spring-Boot-with-Mongodb/)。

本文将会演示如何在Spring Boot项目中使用mysql数据库。


## 1.建立数据库连接（database connection）
在上篇文章中我们新建了一个Spring Boot应用程序，添加了jdbc和data-jpa等starters，以及一个h2数据库依赖，这里我们将配置一个H2数据库。

对于H2、HSQL或者Derby这类嵌入型数据库，只要在pom文件中添加对应的依赖就可以，不需要额外的配置。当spring boot在classpath下发现某个数据库依赖存在且在代码中有关于**Datasource Bean**的定义时，就会自动创建一个数据库连接。我们通过修改之前的bookpub程序说明这个问题，需要修改StartupRunner.java文件，代码如下：

```
package org.test.bookpub;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;

import javax.sql.DataSource;

public class StartupRunner implements CommandLineRunner {
    protected static final Logger logger = LoggerFactory.getLogger(StartupRunner.class);

    @Autowired
    private DataSource ds;

    @Override
    public void run(String... strings) throws Exception {
        logger.info("Datasource: " + ds.toString());
    }
}

```
启动应用程序，可以看到如下输出：driverClassName=org.h2.Driver;因此，可以证明，Spring Boot根据我们自动织入DataSource的代码，自动创建并初始化了一个H2数据库。不过，这个数据库并没什么用，因为存放其中的数据会在系统停止后就丢失。通过修改配置，我们可以将数据存放在磁盘上。关于H2数据库的配置文件如下：

```
spring.datasource.url = jdbc:h2:~/test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.username = sa
spring.datasource.password =
```
然后启动应用程序，并检查你的home目录下是否存在test.mv.db文件。通过“~/test”，就告诉Spring Boot，H2数据库的数据会存放在test.mv.db这个文件中。

综上，可以看出，Spring Boot试图通过spring.datasource分组下的一系列配置项来简化用户对数据库的使用，我们经常使用的配置项有：url，username，password以及driver-class-name等等。PS：driverClassName或者driver-class-name都可以，Spring Boot会在内部进行统一处理。

最常用的开源数据库是Mysql，在Spring Boot通过下列配置项来配置mysql：

```
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/springbootcookbook
spring.datasource.username=root
spring.datasource.password=
```

如果希望通过Hibernate依靠Entity类自动创建数据库和数据表，则还需要加上配置项——`spring.jpa.hibernate.ddl-auto=create-drop`。PS：在生产环境中不要使用create-drop，这样会在程序启动时先删除旧的，再自动创建新的，最好使用update；还可以通过设置`spring.jpa.show-sql = true`来显示自动创建表的SQL语句，通过`spring.jpa.database = MYSQL`指定具体的数据，如果不明确指定Spring boot会根据classpath中的依赖项自动配置。

在Spring项目中，如果数据量比较简单，我们可以考虑使用JdbcTemplate，而不是直接定义Datasource，配置jdbc的代码如下：

```
@Autowired
private JdbcTemplate jdbcTemplate;
```
只要定义了上面这个代码，Spring Boot会自动创建一个Datasource对象，然后再创建一个jdbctemplate对象来管理datasource，通过jdbctemplate操作数据库可以减少大量模板代码。如果你对SpringBoot的原理感兴趣，可以在org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration类中查看其具体实现。

## 2.创建数据仓库服务（data repository service）
连接数据库并直接执行SQL语句这种思路非常古老，早在很多年前就已经出现了ORM（Object Relational Mapping）框架来简化这部分工作，最有名的是Hibernate，但是现在更火的好像是Mybatis。关于spring boot和Mybatis的整合，可以参考：[mybatis-spring-boot](https://github.com/mybatis/mybatis-spring-boot)。我们这里使用Hibernate进行演示。我们将会增加一些实体类，这些实体类决定了数据库的表结构，还要定义一个CrudRepository接口，用于操作数据。

示例程序是一个图书管理系统，显然，数据库中应该具备以下领域对象(domain object)：Book、Author、Reviewrs以及Publisher。首先在src/main/java/org/test/bookpub下建立包domain，然后再在这个包下建立相应的实体类。具体代码列举如下（为了节省空间，省去了getter和setter）：

- Book.java

```
package com.test.bookpub.domain;

import javax.persistence.*;
import java.util.List;

@Entity
public class Book {
    @Id
    @GeneratedValue
    private Long id;
    private String isbn;
    private String title;
    private String description;

    @ManyToOne
    private Author author;
    @ManyToOne
    private Publisher publisher;

    @ManyToMany
    private List<Publisher.Reviewer> reviewers;

    protected Book() { }

    public Book(Author author, String isbn, Publisher publisher, String title) {
        this.author = author;
        this.isbn = isbn;
        this.publisher = publisher;
        this.title = title;
    }
}
```

- Author.java

```
package com.test.bookpub.domain;

import javax.persistence.*;
import java.util.List;

@Entity
public class Author {
    @Id
    @GeneratedValue
    private Long id;
    private String firstName;
    private String lastName;
    @OneToMany(mappedBy = "author")
    private List<Book> books;

    protected Author() {

    }

    public Author(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }
}

```

- Publisher.java

```
package com.test.bookpub.domain;

import javax.persistence.*;
import java.util.List;

@Entity
public class Publisher {
    @Id
    @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "publisher")
    private List<Book> books;

    protected Publisher() { }

    public Publisher(String name) {
        this.name = name;
    }



    @Entity
    public class Reviewer {
        @Id
        @GeneratedValue
        private Long id;
        private String firstName;
        private String lastName;

        protected Reviewer() { }

        public Reviewer(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
    }
}
```
- repository层：创建完实体类，还需要创建BookRepository接口，该接口继承自CrudRepository，这个接口放在src/main/java/com/test/bookpub/repository包中，具体代码如下：

```
package com.test.bookpub.repository;

import com.test.bookpub.domain.Book;
import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface BookRepository extends CrudRepository<Book, Long> {
    Book findBookByIsbn(String isbn);
}

```
- 织入BookRepository，最后需要再StartupRunner中定义BookRepository对象，并自动织入。

```
package com.test.bookpub;

import com.test.bookpub.repository.BookRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;

public class StartupRunner implements CommandLineRunner {
    protected final Logger logger = LoggerFactory.getLogger(StartupRunner.class);

    @Autowired
    private BookRepository bookRepository;

    @Override
    public void run(String... strings) throws Exception {
        logger.info("Number of books: " + bookRepository.count());
    }
}

```

可能读者朋友你也注意到了，到此为止，我们都没有写一行SQL语句，也没有在代码中涉及到数据库连接、建立查询等方面的内容。只有实体类上的各种注解表明我们在于数据库做交互：@Entity，@Repository，@Id，@GeneratedValue，@ManyToOne，@ManyToMany以及@OneToMany，这些注解属于Java Persistance API。我们通过CrudRespository接口的子接口与数据库交互，同时由Spring建立对象与数据库表、数据库表中的数据之间的映射关系。下面依次说明这些注解的含义和使用：

- @Entity，说明被这个注解修饰的类应该与一张数据库表相对应，表的名称可以由类名推断，当然了，也可以明确配置，只要加上`@Table(name = "books")`即可。需要特别注意，每个Entity类都应该有一个protected访问级别的无参构造函数，用于给Hibernate提供初始化的入口。
- @Id and @GeneratedValue：@Id注解修饰的属性应该作为表中的主键处理、@GeneratedValue修饰的属性应该由数据库自动生成，而不需要明确指定。
- @ManyToOne, @ManyToMany表明具体的数据存放在其他表中，在这个例子里，书和作者是多对一的关系，书和出版社是多对一的关系，因此book表中的author和publisher相当于数据表中的外键；并且在Publisher中通过@OneToMany（mapped = "publisher"）定义一个反向关联（1——>n），表明book类中的publisher属性与这里的books形成对应关系。
- @Repository 用来表示访问数据库并操作数据的接口，同时它修饰的接口也可以被component scan机制探测到并注册为bean，这样就可以在其他模块中通过@Autowired织入。
- CrudRepository，直接查看源代码，CrudRepository的代码如下：

```
public interface CrudRepository<T, ID extends Serializable>
    extends Repository<T, ID> {

    <S extends T> S save(S entity); //保存给定的entity

    T findOne(ID primaryKey);//根据给定的id查询对应的entity

    Iterable<T> findAll(); //查询所有entity          

    Long count();//返回entity的个数

    void delete(T entity); //删除给定的entity   

    boolean exists(ID primaryKey); //判断给定id的entity是否存在

    // … more functionality omitted.
}
```
我们可以添加自定义的接口函数，JPA会提供对应的SQL查询，例如，在本例中的BookRepository中可以增加findBookByIsbn(String isbn)函数，JPA会自动创建对应的SQL查询——根据isbn查询图书，这种将方法名转换为SQL语句的机制十分方便且功能强大，例如你可以增加类似findByNameIgnoringCase(String name)这种复杂查询。

最后，我们利用`mvn spring-boot:run`运行应用程序，观察下Hibernate是如何建立数据库连接，如何检测数据表是否存在以及如何自动创建表的过程。

![spring with mysql](http://upload-images.jianshu.io/upload_images/44770-5d281e131d20abdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	

## 3. 参考资料
- [http://docs.spring.io/spring-data/data-commons/docs/current/reference/html/](http://docs.spring.io/spring-data/data-commons/docs/current/reference/html/)
