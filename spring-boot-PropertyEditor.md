# Spring Boot：定制PropertyEditors

在[Spring Boot: 定制HTTP消息转换器](spring-boot-HttpMessageConverters.md)一文中我们学习了如何配置消息转换器用于HTTP请求和响应数据，实际上，在一次请求的完成过程中还发生了其他的转换，我们这次关注将参数转换成多种类型的对象，如：字符串转换成Date对象或字符串转换成Integer对象。

在编写控制器中的action方法时，Spring允许我们使用具体的数据类型定义函数签名，这是通过*PropertyEditor*实现的。*PropertyEditor*本来是JDK提供的API，用于将文本值转换成给定的类型，结果Spring的开发人员发现它恰好满足Spring的需求——将URL参数转换成函数的参数类型。

针对常用的类型（Boolean、Currency和Class），Spring MVC已经提供了很多*PropertyEditor*实现。假设我们需要创建一个Isbn类并用它作为函数中的参数。

## How Do

- 考虑到PropertyEditor属于工具范畴，选择在项目根目录下增加一个包——utils。在这个包下定义Isbn类和IsbnEditor类，各自代码如下：
Isbn类：

```
package com.test.bookpub.utils;

public class Isbn {
    private String isbn;

    public Isbn(String isbn) {
        this.isbn = isbn;
    }
    public String getIsbn() {
        return isbn;
    }
}
```

- IsbnEditor类，继承PropertyEditorSupport类，setAsText完成字符串到具体对象类型的转换，getAsText完成具体对象类型到字符串的转换。

```
package com.test.bookpub.utils;
import org.springframework.util.StringUtils;
import java.beans.PropertyEditorSupport;

public class IsbnEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        if (StringUtils.hasText(text)) {
            setValue(new Isbn(text.trim()));
        } else {
            setValue(null);
        }
    }
    @Override    public String getAsText() {
        Isbn isbn = (Isbn) getValue();
        if (isbn != null) {
            return isbn.getIsbn();
        } else {
            return "";
        }
    }
}
```

- 在BookController中增加initBinder函数，通过@InitBinder注解修饰，则可以针对每个web请求创建一个editor实例。

```
@InitBinderpublic 
void initBinder(WebDataBinder binder) {
    binder.registerCustomEditor(Isbn.class, new IsbnEditor());
}
```

- 修改BookController中对应的函数

```
@RequestMapping(value = "/{isbn}", method = RequestMethod.GET)
public Map<String, Object> getBook(@PathVariable Isbn isbn) {
    Book book =  bookRepository.findBookByIsbn(isbn.getIsbn());
    Map<String, Object> response = new LinkedHashMap<>();
    response.put("message", "get book with isbn(" + isbn.getIsbn() +")");
    response.put("book", book);    return response;
}
```

运行程序，通过Httpie访问`http localhost:8080/books/9781-1234-1111`，可以得到正常结果，跟之前用String表示isbn时没什么不同，说明我们编写的IsbnEditor已经起作用了。

## 分析

Spring提供了很多默认的editor，我们也可以通过继承PropertyEditorSupport实现自己定制化的editor。

由于ProperteyEditor是**非线程安全**的。通过@InitBinder注解修饰的initBinder函数，会为每个web请求初始化一个editor实例，并通过WebDataBinder对象注册。
