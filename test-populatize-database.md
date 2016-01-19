在[Spring Boot应用的测试](test-mockito.md)一文中，我们在*StarterRunner*类的*run(...)*方法中给数据库中添加一些初始数据。尽管通过编程方式添加初始数据比较快捷方便，但长期来看这并不是一个好办法——特别是当需要添加的数据量很大时。我们开发最好把数据库准备、数据库修改和数据库的配置与将要运行的程序代码分离，尽管这仅仅是为测试用例做准备。Spring Boot已经提供了相应的支持来完成这个任务。

我们在之前的应用程序基础上进行实验。Spring Boot提供两种方法来定义数据库的表结构以及添加数据。第一种方法是使用Hibernate提供的工具来创建表结构，该机制会自动搜索@Entity实体对象并创建对应的表，然后使用import.sql文件导入测试数据；第二种方法是利用旧的Spring JDBC，通过schema.sql文件定义数据库的表结构、通过data.sql导入测试数据。

## How Do
- 首先，将现有的“编程式初始化数据”的代码注释掉，因此在StarterRunner中run方法中注释掉下列代码：

```
    @Override
    public void run(String... strings) throws Exception {
//        Author author = new Author("du", "qi");
//        authorRepository.save(author);
//        Publisher publisher = new Publisher("中文测试");
//        publisherRepository.save(publisher);
//        Book book = new Book(author, "9876-5432-1111", publisher, "瞅啥");
//        book.setDescription("23333333,这是一本有趣的书");
//        bookRepository.save(book);
        logger.info("Number of books: " + bookRepository.count());
    }
```
- 在resources目录下新建*import.sql*文件(注意，SQL语句中指定的字段要与Hibernate自动生成的表的字段相同)，该文件的内容如下：

```
INSERT INTO author (id, first_name, last_name) VALUES (1, "Alex", "Antonov");
INSERT INTO publisher (id, name) VALUES (1, "Packt");
INSERT INTO book (isbn, title, author, publisher) VALUES ("9876-5432-1111", "Spring Boot Recipes", 1,1);
```
- 现在运行测试用例，发现可以通过;
- 第二种方法是获取Spring JDBC的支持，需要我们提供schema.sql和data.sql文件。现在可以将import.sql重命名为data.sql，然后再创建新的文件schema.sql。在删除数据表时，需要考虑依赖关系，例如表A依赖表B，则先删除表B。创建数据库关系的内容如下：

```
-- clear context
DROP TABLE IF EXISTS `book_reviewers`;
DROP TABLE IF EXISTS `reviewer`;
DROP TABLE IF EXISTS `book`;
DROP TABLE IF EXISTS `author`;
DROP TABLE IF EXISTS `publisher`;

-- Create syntax for TABLE 'author'
CREATE TABLE `author` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
  `first_name` VARCHAR(255) DEFAULT NULL ,
  `last_name` VARCHAR(255) DEFAULT NULL ,
  PRIMARY KEY (`id`)
);

-- Create syntax for TABLE 'publisher'
CREATE TABLE `publisher` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) DEFAULT NULL ,
  PRIMARY KEY (`id`)
);

-- Create syntax for TABLE 'book'
CREATE TABLE `book` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
  `description` VARCHAR(255) DEFAULT NULL ,
  `isbn` VARCHAR(255) DEFAULT NULL ,
  `title` VARCHAR(255) DEFAULT NULL ,
  `author` BIGINT(20) DEFAULT NULL ,
  `publisher` BIGINT(20) DEFAULT NULL ,
  PRIMARY KEY (`id`),
  CONSTRAINT `FK_publisher` FOREIGN KEY (`publisher`) REFERENCES `publisher` (`id`),
  CONSTRAINT `FK_author` FOREIGN KEY (`author`) REFERENCES `author` (`id`)
);

-- Create syntax for TABLE 'reviewer'
CREATE TABLE `reviewer` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
  `first_name` VARCHAR(255) DEFAULT NULL ,
  `last_name` VARCHAR(255) DEFAULT NULL ,
  PRIMARY KEY (`id`)
);

-- Create syntax for TABLE 'book_reviewers'
CREATE TABLE `book_reviewers` (
  `books` BIGINT(20) NOT NULL ,
  `reviewers` BIGINT(20) NOT NULL ,
  CONSTRAINT `FK_book` FOREIGN KEY (`books`) REFERENCES `book` (`id`),
  CONSTRAINT `FK_reviewer` FOREIGN KEY (`reviewers`) REFERENCES `reviewer` (`id`)
);
```
- 我们手动创建了数据库表结构，因此需要关掉Hibernate的自动创建开关，即在*application.properties*中设置*spring.jpa.hibernate.ddl-auto = none*
- 运行测试，发现测试可以正常通过。

> Note：个人建议是使用Hibernate的自动创建机制，当然这会少一点可定制性；最近更流行的是Mybatis，mybatis-spring-boot也可以使用，mybatis的可定制性更强。

## 分析
在Spring社区中常常可以通过使用各种组件，例如Spring JDBC、Spring JPA with Hibernate，或者Flyway、Liquidbase这类数据库迁移工具，都能实现类似的功能。

> Note：Flyway和Liquidbase都提供数据库的增量迁移功能。当项目中需要管理数据库的增量变动，并且需要快速切换到指定的数据版本时，非常适合使用Flyway和Liquidbase，更多的信息可以参考[http://flywaydb.org/](http://flywaydb.org/)和[http://www.liquibase.org/](http://www.liquibase.org/)。

在上文中我们使用了两种不同的方法来初始化数据库和填充测试数据

### 使用Spring JPA with Hibernate初始化数据库
这种方法中，由*Hibernate*库完成大部分工作，我们只需要配置合适的配置项，并创建对应的实体类的定义。在这个方案中我们主要使用以下配置项：
- *spring.jpa.hibernate.ddl-auto=create-drop*配置项告诉Hibernate通过*@Entity*模型的定义自动推断数据库定义并创建合适的表。在程序启动时，经由Hibernate计算出的schema会用来创建表结构，在程序结束时这些表也被删除。即使程序强制退出或者奔溃，在重新启动的时候也会先把之前的表删除，并重新创建——因此"create-drop"这种配置不适合生产环境。PS:如果程序没有显式配置*spring.jpa.hibernate.ddl-auto*属性，Spring Boot会给H2这类的嵌入式数据库配置create-drop，因此需要仔细斟酌这个配置项。

- 在classpath下创建*import.sql*文件供*Hibernate*使用，该文件中的内容是一些SQL语句，将会在应用程序启动时运行。尽管该文件中可以写任何有效的SQL语句，不过建议只写数据操作语句，例如INSERT、UPDATE等等。

### 使用Spring JDBC初始化数据库
如果项目中没有用JPA或者你不想依赖Hibernate库，Spring提供另外一种方法来设置数据库，当然，首先需要提供*spring-boot-starter-jdbc*依赖。

- *spring.jpa.hibernate.ddl-auto=none*表示Hibernate不会自动创建数据库表结构。在生产环境中最好用这个设置，能够避免你不小心将数据库全部删除（那一定是一个噩梦）。

- *schema.sql*文件包含创建数据库表结构的SQL语句，在应用程序启动过程中，需要创建数据库表结构时，执行该文件中的DDL语句。Hibernate会自动删除已经存在的表，如果我们希望只有某个表不存在的时候才创建它，可以在这个文件开头最好先使用`DROP TABLE IF EXISTS`删除可能存在的表，再使用`CREATE TABLE IF NOT EXISTS`创建表。这种用法可以灵活得定义数据库中的表结构，因此在生产环境中用更安全。

- *data.sql*的作用跟上一个方法的*import.sql*一样，用于存放数据导入的SQL语句。

考虑到这是Spring的特性，我们可以不只是全局定义数据库定义文件，还可以针对不同的数据库定义不同的文件。例如，可以定义给Oracle数据库使用的schema-oracle.sql，给MySQL数据库用的schema-mysql.sql文件；对于data.sql文件，则可以由不同数据库共用。如果你希望覆盖Spring Boot的自动推断，可以配置*spring.datasource.platform*属性。

>Tip：如果你希望使用别的名字代替schema.sql或者data.sql，Spring Boot也提供了对应的配置属性，即*spring.datasource.schema*和*spring.datasource.data*。
