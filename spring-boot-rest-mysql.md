# RESTful by Spring Boot with MySQL

现在的潮流是前端承担越来越多的责任：MVC中的V和C，后端只需要负责提供数据M，但是后端有更重要的任务：高并发、提供各个维度的扩展能力（负载均衡、数据表切分、服务分离）、更清晰的API设计。Spring Boot框架提供的机制便于工程师实现标准的RESTful接口，本文主要讨论如何编写Controller代码，另外还涉及了MySQL的数据库操作，之前我也写过一篇关于Mysql的文章[link](http://www.jianshu.com/p/1b626a6f550e)，但是这篇文章加上了CRUD的操作。

先回顾下之前的文章中我们用到的例子：图书信息管理系统，主要的领域对象有book、author、publisher和reviewer。

首先我们要在pom文件中添加对应的starter，即`spring-boot-starter-web`，对应的xml代码示例为：

```
<dependency>   
     <groupId>org.springframework.boot</groupId>   
     <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

然后我们要创建控制器（Controller），先在项目根目录下创建controller包，一般为每个实体类对象创建一个控制器，例如BookController。

@RestController注解是@Controller和@ResponseBody的合集，表示这是个控制器bean，并且是将函数的返回值直接填入HTTP响应体中，是REST风格的控制器。@RequestMapping("/books")表示该控制器处理所有“/books”的URL请求，具体由那个函数处理，要根据HTTP的方法来区分：GET表示查询、POST表示提交、PUT表示更新、DELETE表示删除。

- 查询所有图书记录：利用@Autowired导入BookRepository的Bean，直接调用bookRepository.findAllBooks()即可。我们的返回值形式如下。关于RESTful返回值形式的设计，后续会有专门的文章讨论。

```
{
    "message": "get all books",
    "book": [
        {
            "isbn": "9781-1234-5678",
            "title": "你愁啥",
            "description": "这是一本奇怪的书",
            "author": {
                "firstName": "冯",
                "lastName": "pp"
            },
            "publisher": {
                "name": "大锤出版社"
            },
            "reviewers": []
        },
        {
            "isbn": "9781-1234-1111",
            "title": "别吵吵",
            "description": "哈哈哈",
            "author": {
                "firstName": "杜琪",
                "lastName": "琪"
            },
            "publisher": {
                "name": "大锤出版社"
            },
            "reviewers": []
        }
    ]
}
```
- 根据isbn查询图书记录：根据isbn查询一本书的记录，调用bookRepository.findBookByIsbn()即可。返回值形式如下：

```
{
    "message": "get book with isbn(9781-1234-5678)",
    "book": {
        "isbn": "9781-1234-5678",
        "title": "你愁啥",
        "description": "这是一本奇怪的书",
        "author": {
            "firstName": "冯",
            "lastName": "pp"
        },
        "publisher": {
            "name": "大锤出版社"
        },
        "reviewers": []
    }
}
```

- 添加图书记录，客户端的图书信息封装成json字符串传递过来，因此利用@RequestBody获取POST请求体，由于book记录中有外链记录，因此要首先解析出author对象和publisher对象，并将它们存入数据库；然后才生成book对象，并调用bookRepository.save(book)将book记录存入数据库。该接口的返回值会把刚添加的图书信息返回给客户端，形式类似于getBookByIsbn这个接口。
- 更新图书书名，这里简单以这个接口作为更新的例子。主要步骤是先取出对应isbn的book对象，然后`book.setTitle(title)`更新book信息，然后调用bookRepository.save(book)更新该对象的信息，通过@PathVariable修饰的参数title与URL中用“{title}”的值对应。
- 删除图书记录；给定图书的isbn直接删除即可。

最后，放上完整的Controller代码：

```
package com.test.bookpub.controller;

import com.alibaba.fastjson.JSONObject;
import com.test.bookpub.domain.Author;
import com.test.bookpub.domain.Book;
import com.test.bookpub.domain.Publisher;
import com.test.bookpub.repository.AuthorRepository;
import com.test.bookpub.repository.BookRepository;
import com.test.bookpub.repository.PublisherRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.*;

/**
 * @author duqi
 * @create 2015-12-02 18:18
 */
@RestController
@RequestMapping("/books")
public class BookController {
    private static final Logger logger = LoggerFactory.getLogger(BookController.class);

    @Autowired
    private BookRepository bookRepository;
    @Autowired
    public AuthorRepository authorRepository;
    @Autowired
    public PublisherRepository publisherRepository;


    @RequestMapping(method = RequestMethod.GET)
    public Iterable<Book> getAllBooks() {
        return bookRepository.findAll();
    }

    @RequestMapping(value = "/{isbn}", method = RequestMethod.GET)
    public Map<String, Object> getBook(@PathVariable String isbn) {
        Book book =  bookRepository.findBookByIsbn(isbn);

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("message", "get book with isbn(" + isbn +")");
        response.put("book", book);
        return response;
    }

    @RequestMapping(method = RequestMethod.POST)
    public Map<String, Object> addBook(@RequestBody JSONObject bookJson) {
        JSONObject authorJson = bookJson.getJSONObject("author");
        Author author = new Author(authorJson.getString("firstName"), authorJson.getString("lastName"));
        authorRepository.save(author);
        String isbn = bookJson.getString("isbn");
        JSONObject publisherJson = bookJson.getJSONObject("publisher");
        Publisher publisher = new Publisher(publisherJson.getString("name"));
        publisherRepository.save(publisher);
        String title = bookJson.getString("title");
        String desc = bookJson.getString("desc");
        Book book = new Book(author, isbn, publisher, title);
        book.setDescription(desc);
        bookRepository.save(book);

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("message", "book add successfully");
        response.put("book", book);
        return response;
    }

    @RequestMapping(value = "/{isbn}", method = RequestMethod.DELETE)
    public Map<String, Object> deleteBook(@PathVariable String isbn) {
        Map<String, Object> response = new LinkedHashMap<>();
        try {
            bookRepository.deleteBookByIsbn(isbn);
        } catch (NullPointerException e) {
            logger.error("the book is not in database");
            response.put("message", "delete failure");
            response.put("code", 0);
        }

        response.put("message", "delete successfully");
        response.put("code", 1);
        return response;
    }

    @RequestMapping(value = "/{isbn}/{title}", method = RequestMethod.PUT)
    public Map<String, Object> updateBookTitle(@PathVariable String isbn, @PathVariable String title) {
        Map<String, Object> response = new LinkedHashMap<>();
        Book book = null;
        try {
            book = bookRepository.findBookByIsbn(isbn);
            book.setTitle(title);
            bookRepository.save(book);
        } catch (NullPointerException e) {
            response.put("message", "can not find the book");
            return response;
        }

        response.put("message", "book update successfully");
        response.put("book", book);
        return response;
    }
}
```

*有三个问题需要补充探讨*

现在我要说下Controller的角色，大家可以看到，我这里将很多业务代码混淆在Controller的代码中。实际上，根据[程序员必知之前端演进史](http://segmentfault.com/a/1190000003973286)一文所述Controller层应该做的事是：  处理请求的参数 渲染和重定向 选择Model和Service 处理Session和Cookies，我基本上认同这个观点，最多再加上OAuth验证（利用拦截器实现即可）。而真正的业务逻辑应该单独分处一层来处理，即常见的service层；

今天遇到一个类似参考资料2中的错误，我经过查找后发现是Jakson解析我的对象的时候出现了无限递归解析，究其原因，是因为外链：解析book的时候，需要解析author，但是在author中又有books选项，所以造成死循环，解决的办法就是在author中的books属性上加上注解：@JsonBackReference；同样需要在Publisher类中的books属性加上@JsonBackReference注解。

上述演示的Controller代码还有两个问题：返回值形式不统一；并没有遵循标准的API设计（例如update方法实际上应该由客户端返回更新过的完整对象，这样就可以直接调用save方法），后续，我会参考[RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)进行学习，对API的设计进行自己的学习总结，读者朋友，你也需要自己实践和学习哦，有问题的可以找我讨论。

## 参考资料

1. [repository中的update方法](http://stackoverflow.com/questions/24420572/update-or-saveorupdate-in-crudrespository-is-there-any-options-available)
2. [使用spring data创建REST应用](https://crate.io/blog/using-sprint-data-crate-with-your-java-rest-application/)
3. [遇到的一个错误：at com.fasterxml.jackson.databind.ser.BeanSerializer.serialize](http://stackoverflow.com/questions/23055801/jackson-annotations-not-used-with-spring-boot)
4. [SPRING BOOT: DATA ACCESS WITH JPA, HIBERNATE AND MYSQL](http://blog.netgloo.com/2014/10/06/spring-boot-data-access-with-jpa-hibernate-and-mysql/)
