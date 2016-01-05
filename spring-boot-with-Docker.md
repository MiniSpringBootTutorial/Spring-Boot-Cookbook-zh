前段时间在我厂卷爷的指导下将Docker在我的实际项目中落地，最近几个小demo都尽量熟悉docker的使用，希望通过这篇文章分享我截止目前的使用经验（如有不准确的表述，欢迎帮我指出）。本文的主要内容是关于Java应用程序的docker化，首先简单介绍了docker和docker-compose，然后利用两个案例进行实践说明。

简单说说Docker，现在云计算领域火得一塌糊涂的就是它了吧。Docker的出现是为了解决PaaS的问题：运行环境与具体的语言版本、项目路径强关联，因此干脆利用lxc技术进行资源隔离，构造出跟随应用发布的运行环境，这样就解决了语言版本的限制问题。PaaS的出现是为了让运维人员不需要管理一台虚拟机，IaaS的出现是为了让运维人员不需要管理物理机。云计算，说到底都是俩字——运维。

云计算领域的技术分为虚拟化技术和资源管理两个方面，正好对应我们今天要讲的两个工具：Docker和docker-compose。Docker的主要概念有：容器、镜像、仓库；docker-compose是fig的后续版本，负责将多个docker服务整合起来，对外提供一致服务。

## 1. Spring Boot应用的docker化
首先看Spring Boot应用程序的docker化，由于Spring Boot内嵌了tomcat、Jetty等容器，因此我们对docker镜像的要求就是需要java运行环境。我的应用代码的的Dockerfile文件如下：

```
#基础镜像：仓库是java，标签用8u66-jdk
FROM java:8u66-jdk
#当前镜像的维护者和联系方式
MAINTAINER duqi duqi@example.com
#将打包好的spring程序拷贝到容器中的指定位置
ADD target/bookpub-0.0.1-SNAPSHOT.jar /opt/bookpub-0.0.1-SNAPSHOT.jar
#容器对外暴露8080端口
EXPOSE 8080
#容器启动后需要执行的命令
CMD java -Djava.security.egd=file:/dev/./urandom -jar /opt/bookpub-0.0.1-SNAPSHOT.jar
```
因为目前的示例程序比较简单，这个dockerfile并没有在将应用程序的数据存放在宿主机上。如果你的应用程序需要写文件系统，例如日志，最好利用`VOLUME /tmp`命令，这个命令的效果是：在宿主机的/var/lib/docker目录下创建一个临时文件并把它链接到容器中的/tmp目录。

把这个Dockerfile放在项目的根目录下即可，后续通过`docker-compose build`统一构建：基础镜像是只读的，然后会在该基础镜像上增加新的可写层来供我们使用，因此java镜像只需要下载一次。

**docker-compose**是用来做docker服务编排，参看《Docker从入门到实践》中的解释：
>Compose 项目目前在 Github 上进行维护，目前最新版本是 1.2.0。Compose 定位是“defining and running complex applications with Docker”，前身是 Fig，兼容 Fig 的模板文件。

>Dockerfile 可以让用户管理一个单独的应用容器；而 Compose 则允许用户在一个模板（YAML 格式）中定义一组相关联的应用容器（被称为一个 project，即项目），例如一个 Web 服务容器再加上后端的数据库服务容器等。

单个docker用起来确实没什么用，docker技术的关键在于持续交付，通过与jekins的结合，可以实现这样的效果：开发人员提交push，然后jekins就自动构建并测试刚提交的代码，这就是我理解的持续交付。

## 2. spring boot + redis + mongodb
在这个项目中，我启动三个容器：web、redis和mongodb，然后将web与redis连接，web与mongodb连接。首先要进行redis和mongodb的docker化，redis镜像的Dockerfile内容是：

```
FROM        ubuntu:14.04
RUN         apt-get update
RUN         apt-get -y install redis-server
EXPOSE      6379
ENTRYPOINT  ["/usr/bin/redis-server"]
```
Mongodb镜像的Dockerfile内容是,docker官方给了mongodb的docker化教程，我直接拿来用了，参见[Dockerizing MongoDB](https://docs.docker.com/engine/examples/mongodb/)：

```
# Format: FROM repository[:version]
FROM       ubuntu:14.04
# Format: MAINTAINER Name <email@addr.ess>
MAINTAINER duqi duqi@example.com
# Installation:
# Import MongoDB public GPG key AND create a MongoDB list file
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo "deb http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-3.0.list
# Update apt-get sources AND install MongoDB
RUN apt-get update && apt-get install -y mongodb-org
# Create the MongoDB data directory
RUN mkdir -p /data/db
# Expose port 27017 from the container to the host
EXPOSE 27017
# Set usr/bin/mongod as the dockerized entry-point application
ENTRYPOINT ["/usr/bin/mongod"]
```

使用docker-compose编排三个服务，具体的模板文件如下：
```
web:
  build: .
  ports:
   - "49161:8080"
  links:
   - redis
   - mongodb
   
redis:
    image: duqi/redis
    ports:
      - "6379:6379"

mongodb:
    image: duqi/mongodb
    ports:
      - "27017:27017"
```

架构比较简单，第一个区块的build，表示docker中的命令“docker build .”，用于构建web镜像；ports这块表示将容器的8080端口与宿主机（IP地址是：192.168.99.100）的49161对应。因为现在docker不支持原生的osx，因此在mac下使用docker，实际上是在mac上的一台虚拟机（docker-machine）上使用docker，这台机器的地址就是192.168.99.100。参见：[在mac下使用docker](https://docs.docker.com/v1.8/installation/mac/)

links表示要连接的服务，redis与下方的redis区块对应、mongodb与下方的mongodb区块对应。redis和mongodb类似，首先说明要使用的镜像，然后规定端口映射。

那么，如何运行呢？
1. 命令`docker-compose build`，表示构建web服务，目前我用得比较土，就是编译jar之后还需要重新更新docker，优雅点不应该这样。
![docker-compose build](http://upload-images.jianshu.io/upload_images/44770-feb314e36b94f1ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. 命令`docker-compose up`，表示启动web服务，可以看到mongodb、redis和web依次启动，启动后用`docker ps`查看当前的运行容器。
![docker ps](http://upload-images.jianshu.io/upload_images/44770-97dc7650eadaa9ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

特别注意，在配置文件中写redis和mongodb的url时，要用虚拟机的地址，即192.168.99.100。例如，redis的一个配置应该为：spring.redis.host=192.168.99.100。

## 3. spring boot + mysql
拉取mysql镜像的指令是：`docker run --name db001 -p 3306:3306  -e MYSQL_ROOT_PASSWORD=admin -d mysql:5.7`，表示启动的docker容器名字是db001，登录密码一定要设定， -d表示设置Mysql版本。

我的docker-compose模板文件是：

```
web:
  build: .
  ports:
   - "49161:8080"
  links:
   - mysql

mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MYSQL_DATABASE: springbootcookbook
    ports:
      - "3306:3306"
```

主要内容跟之前的类似，主要讲下mysql部分，通过environement来设置进入mysql容器后的环境变量，即连接数据库的密码MYSQL_ROOT_PASSWORD，使用的数据库名称MSYQL_DATABASE等等。

一直想写这篇文章做个总结，写来发现还是有点薄，对于docker我还需要系统得学习，不过，针对上面的例子，我都是亲自实践过的，大家有什么问题可以与我联系。

## 参考资料
1. [Docker从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice/details)
2. [Docker - Initialize mysql database with schema](http://stackoverflow.com/questions/29145370/docker-initialize-mysql-database-with-schema)
3. [使用Docker搭建基础的mysql应用](http://blog.csdn.net/smallfish1983/article/details/40080305)
4. [Spring Boot with docker](https://spring.io/guides/gs/spring-boot-docker/)
