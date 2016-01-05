考虑到*PropertyEditor*的无状态和非线程安全特性，Spring 3增加了一个*Formatter*接口来替代它。Formatters提供和PropertyEditor类似的功能，但是提供线程安全特性，并且专注于完成字符串和对象类型的互相转换。

假设在我们的程序中，需要根据一本书的ISBN字符串得到对应的book对象。通过这个类型格式化工具，我们可以在控制器的方法签名中定义Book参数，而URL参数只需要包含ISBN号和数据库ID。

## How Do
-  首先在项目根目录下创建*formatters*包
-  然后创建BookFormatter，它实现了Formatter接口，实现两个函数：parse用于将字符串ISBN转换成book对象；print用于将book对象转换成该book对应的ISBN字符串。

```
package com.test.bookpub.formatters;

import com.test.bookpub.domain.Book;
import com.test.bookpub.repository.BookRepository;
import org.springframework.format.Formatter;
import java.text.ParseException;
import java.util.Locale;

public class BookFormatter implements Formatter<Book> {
    private BookRepository repository;

    public BookFormatter(BookRepository repository) {
        this.repository = repository;
    }
    @Override
    public Book parse(String bookIdentifier, Locale locale) throws ParseException {
        Book book = repository.findBookByIsbn(bookIdentifier);
        return book != null ? book : repository.findOne(Long.valueOf(bookIdentifier));
    }
    @Override
    public String print(Book book, Locale locale) {
        return book.getIsbn();
    }
}
```
- 在WebConfiguration中添加我们定义的formatter，重写（@Override修饰）*addFormatter(FormatterRegistry registry)*函数。

```
@Autowired
private BookRepository bookRepository;
@Override
public void addFormatters(FormatterRegistry registry) {
    registry.addFormatter(new BookFormatter(bookRepository));
}
```
- 最后，需要在BookController中新加一个函数getReviewers，根据一本书的ISBN号获取该书的审阅人。

```
@RequestMapping(value = "/{isbn}/reviewers", method = RequestMethod.GET)
public List<Reviewer> getReviewers(@PathVariable("isbn") Book book) {
    return book.getReviewers();
}
```
- 通过`mvn spring-boot:run`运行程序
- 通过httpie访问URL——http://localhost:8080/books/9781-1234-1111/reviewers，得到的结果如下：

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Date: Tue, 08 Dec 2015 08:15:31 GMT
Server: Apache-Coyote/1.1
Transfer-Encoding: chunked

[]

```
## 分析
*Formatter*工具的目标是提供跟PropertyEditor类似的功能。通过FormatterRegistry将我们自己的formtter注册到系统中，然后Spring会自动完成文本表示的book和book实体对象之间的互相转换。由于Formatter是无状态的，因此不需要为每个请求都执行注册formatter的动作。

**使用建议：**如果需要通用类型的转换——例如String或Boolean，最好使用PropertyEditor完成，因为这种需求可能不是全局需要的，只是某个Controller的定制功能需求。

我们在WebConfiguration中引入（@Autowired）了*BookRepository*（需要用它创建BookFormatter实例），Spring给配置文件提供了使用其他bean对象的能力。Spring本身会确保BookRepository先创建，然后在WebConfiguration类的创建过程中引入。
