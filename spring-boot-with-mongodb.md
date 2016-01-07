# Restful: Spring Boot with Mongodb

## 为什么是mongodb?

继续之前的dailyReport项目，今天的任务是选择mongogdb作为持久化存储。

关于nosql和rdbms的对比以及选择，我参考了不少资料，关键一点在于：nosql可以轻易扩展表的列，对于业务快速变化的应用场景非常适合；rdbms则需要安装关系型数据库模式对业务进行建模，适合业务场景已经成熟的系统。我目前的这个项目——dailyReport，我暂时没法确定的是，对于一个report，它的属性应该有哪些：date、title、content、address、images等等，基于此我选择mongodb作为该项目的持久化存储。

## 如何将mongodb与spring boot结合使用

- 修改Pom文件，增加mongodb支持

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

- 重新设计Report实体类，id属性是给mongodb用的，用@Id注解修饰；重载toString函数，使用String.format输出该对象。

```
import org.springframework.data.annotation.Id;

/**
 * @author duqi
 * @create 2015-11-17 19:31
 */
public class Report {

    @Id
    private String id;

    private String date;
    private String content;
    private String title;

    public Report() {

    }

    public Report(String date, String title, String content) {
        this.date = date;
        this.title = title;
        this.content = content;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getDate() {
        return date;
    }

    public void setDate(String dateStr) {
        this.date = dateStr;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    @Override
    public String toString() {
        return String.format("Report[id=%s, date='%s', content='%s', title='%s']", id, date, content, title);
    }
}
```

- 增加ReportRepository接口，它继承自MongoRepository接口，MongoRepository接口包含了常用的CRUD操作，例如：save、insert、findall等等。我们可以定义自己的查找接口，例如根据report的title搜索，具体的ReportRepository接口代码如下：

```
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.List;

/**
 * Created by duqi on 15/11/22.
 */
public interface ReportRepository extends MongoRepository<Report, String> {
    Report findByTitle(String title);
    List<Report> findByDate(String date);
}
```

- 修改ReportService代码，增加createReport函数，该函数根据Conroller传来的Map参数初始化一个Report对象，并调用ReportRepository将数据save到mongodb中；对于getReportDetails函数，仍然开启缓存，如果没有缓存的时候则利用findByTitle接口查询mongodb数据库。

```
import com.javadu.dailyReport.domain.Report;
import com.javadu.dailyReport.domain.ReportRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.Map;

/**
 * @author duqi
 * @create 2015-11-17 20:05
 */

@Service
public class ReportService {

    @Autowired
    private ReportRepository repository;

    public Report createReport(Map<String, Object> reportMap) {
        Report report = new Report(reportMap.get("date").toString(),
                reportMap.get("title").toString(),
                reportMap.get("content").toString());

        repository.save(report);
        return report;
    }

    @Cacheable(value = "reportcache", keyGenerator = "wiselyKeyGenerator")
    public Report getReportDetails(String title) {
        System.out.println("无缓存的时候调用这里---数据库查询, title=" + title);
        return repository.findByTitle(title);
    }
}
```

## Restful接口

Controller只负责URL到具体Service的映射，而在Service层进行真正的业务逻辑处理，我们这里的业务逻辑异常简单，因此显得Service层可有可无，但是如果业务逻辑复杂起来（比方说要通过RPC调用一个异地服务），这些操作都需要再service层完成。总体来讲，Controller层只负责：转发请求 + 构造Response数据；在需要进行权限验证的时候，也在Controller层利用aop完成。

一般将对于Report（某个实体）的所有操作放在一个Controller中，并用@RestController和@RequestMapping("/report")注解修饰,表示所有xxxx/report开头的URL会由这个ReportController进行处理。

. **POST**

对于增加report操作，我们选择POST方法，并使用@RequestBody修饰POST请求的请求体，也就是createReport函数的参数；

. **GET**

对于查询report操作，我们选择GET方法，URL的形式是：“xxx/report/${report's title}”，使用@PathVariable修饰url输入的参数，即title。

```
import com.javadu.dailyReport.domain.Report;
import com.javadu.dailyReport.service.ReportService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * @author duqi
 * @create 2015-11-17 20:10
 */

@RestController
@RequestMapping("/report")
public class ReportController {
    private static final Logger logger = LoggerFactory.getLogger(ReportController.class);

    @Autowired
    ReportService reportService;

    @RequestMapping(method = RequestMethod.POST)
    public Map<String, Object> createReport(@RequestBody Map<String, Object> reportMap) {
        logger.info("createReport");
        Report report = reportService.createReport(reportMap);

        Map<String, Object> response = new LinkedHashMap<String, Object>();
        response.put("message", "Report created successfully");
        response.put("report", report);

        return response;
    }

    @RequestMapping(method = RequestMethod.GET, value = "/{reportTitle}")
    public Report getReportDetails(@PathVariable("reportTitle") String title) {
        logger.info("getReportDetails");
        return reportService.getReportDetails(title);
    }
}
```

Update和delete操作我这里就不一一讲述了，留个读者作为练习

## 参考资料

1. [sql vs nosql: what you need to know](http://dataconomy.com/sql-vs-nosql-need-know/)
2. [Accessing data with Mongodb](https://spring.io/guides/gs/accessing-data-mongodb/)
3. [Spring Boot：Restful API using Spring Boot and Mongodb](http://www.javabeat.net/restful-springboot-mongodb/)
