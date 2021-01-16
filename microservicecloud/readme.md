# 前前言

## 背景

这个项目是我根据java3y的进行修改的，本来想直接用的，但是因为我的jdk版本是12，只能用SpringBoot2.x以上，所以对应的Spring Cloud版本也会有很大的不同，刚开始配环境就花了我不少时间接下来讲一下我遇到的坑。

**以下内容都是我解决问题的过程，这些问题在新版中都已经不存在了。**

## 版本对应关系

刚开始配环境就花了不少时间，特别是各个组件的版本对应关系，以及更新版本之后的改动。

具体的版本对应关系比较多，可以参考这篇文章[JDK、Mybatis、Mysql、Maven、Spring Boot以及Spring Cloud的版本对应关系](https://blog.csdn.net/weixin_42972730/article/details/112723830)

## 工作流程

1. 启动Eureka Server
   首先是将服务注册到Eureka中，才能够被其他服务发现
   https://www.lagou.com/lgeduarticle/15243.html
   配置信息可以参考EurekaInstanceConfigBean和EurekaClientConfigBean两个配置类
2. 将服务的提供者注册到Eureka中
   通常需要对数据库和实体类进行操作，不可能每个服务都是单独的实体类，所以怎么将实体类传过去的呢？直接导入module就行了，所以要先将需要的module进行打包（我这里是 api 模块，没有启动类，怎么进行打包呢？）
3. 启动Ribbon进行负载均衡
   如果不启动是否可行呢？这个主要是看是否有耦合性，Ribbon是在哪里进行处理，我认为是对Eureka中进行处理，也就是接收到请求之后，从Eureka中选择相应的服务进行提供
   Ribbon就在服务消费者那里配置的

## maven导入其他模块

https://blog.csdn.net/lijinzhou2017/article/details/78962257

maven 兄弟module间依赖打包报错"Could not find artifact xxx"

https://blog.csdn.net/beijihukk/article/details/104303651

https://blog.csdn.net/FLL430/article/details/104488622

### Spring Cloud 的Mapper中的实体类应该怎么写呢

可以引入带有实体类的模块

## MySQL 5.x与8.x的驱动

https://blog.csdn.net/qq_38343032/article/details/105425935

5.x：org.gjt.mm.mysql.Driver

8.x：com.mysql.jdbc.Driver

org.gjt.mm.mysql.Driver 是com.mysql.jdbc.Driver的前身

## Spring Cloud的运作流程

消费者只是一个调用后端的接口，后端的具体实现是在生产中的，如果我要将项目重构为Spring Cloud的，首先将整个项目作为一个服务，然后再将服务一个个的拆分出去

## 启动hystrix报错

![image-20210116150301687](https://gitee.com/hzm_pwj/FigureBed/raw/master/giteeImg/20210116150301.png)

这个时候不用管，可以看到过一会儿自己就好了。。。

解决方法是在配置文件中添加一下内容，从而不向eureka注册自己

```
server:
  port: 9002


eureka:
  client:
    register-with-eureka: false  # 当前微服务不注册到eureka中(消费端)
    service-url:
      defaultZone: http://eureka7001:7001/eureka/,http://eureka7002:7002/eureka/,http://eureka7003:7003/eureka/
```

## 解决Hystrix Dashboard出现Unable to connect to Command Metric Stream错误

在新版的Spring Cloud中，需要添加Serverlet才能实现信息采集(还有一个更好的方式是不需要配置Bean的，但是我找不到那个方法了，理论上来讲，可以从源码中找出来)

```java
@Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/actuator/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
```

还有就是在dashboard的yml中添加如下内容，以运行相关配置

```yml
hystrix:
  dashboard:
    proxy-stream-allow-list: "*"
```

# 前言 #

该Demo来源于网上，我为其增添了注释和使用手册..

# 一、准备工作 #

## 1.1配置虚拟主机映射 ##

hosts文件路径：`C:\Windows\System32\drivers\etc`

增加三句：

原版是xxx.com，但是不建议用.com之类的后缀，不然回去DNS服务器找对应的域名进行解析，从而导致不能识别

```
127.0.0.1 eureka7001
127.0.0.1 eureka7002
127.0.0.1 eureka7003
127.0.0.1 config-3344
```

## 1.2数据库 ##

创建三个数据库

```sql

create database cloudDB01;
create database cloudDB02;
create database cloudDB03;

```

三个库都创建相同的表


```sql


CREATE TABLE Dept
(
    deptno INT PRIMARY KEY,
    dname VARCHAR(50),
    db_source VARCHAR(50)
);

```

为表插入数据,db_source的值根据1,2,3**分别标识**(cloudDB01库对应的是1,cloudDB02库对应的是2)

```sql
-- database 1

INSERT INTO dept (deptno, dname, db_source) VALUES (1, 'Java1y', '1');
INSERT INTO dept (deptno, dname, db_source) VALUES (2, 'Java2y', '1');
INSERT INTO dept (deptno, dname, db_source) VALUES (3, 'Java3y', '1');
INSERT INTO dept (deptno, dname, db_source) VALUES (4, 'Java4y', '1');
INSERT INTO dept (deptno, dname, db_source) VALUES (5, 'Java5y', '1');

-- database 2


INSERT INTO dept (deptno, dname, db_source) VALUES (1, 'Java1y', '2');
INSERT INTO dept (deptno, dname, db_source) VALUES (2, 'Java2y', '2');
INSERT INTO dept (deptno, dname, db_source) VALUES (3, 'Java3y', '2');
INSERT INTO dept (deptno, dname, db_source) VALUES (4, 'Java4y', '2');
INSERT INTO dept (deptno, dname, db_source) VALUES (5, 'Java5y', '2');

-- database 3


INSERT INTO dept (deptno, dname, db_source) VALUES (1, 'Java1y', '3');
INSERT INTO dept (deptno, dname, db_source) VALUES (2, 'Java2y', '3');
INSERT INTO dept (deptno, dname, db_source) VALUES (3, 'Java3y', '3');
INSERT INTO dept (deptno, dname, db_source) VALUES (4, 'Java4y', '3');
INSERT INTO dept (deptno, dname, db_source) VALUES (5, 'Java5y', '3');
```

# 二、模块之间解释 #


# 2.0通用API #

创建Dept实体，各个模块都可以使用了(不用每个微服务都创建一个Dept对象)。同时Service接口，Feign的代码也在那里编写。

- microservicecloud-api

# 2.1服务注册中心 #


Eureka服务注册中心(集群)

- microservicecloud-eureka-7001
- microservicecloud-eureka-7002
- microservicecloud-eureka-7003


## 2.2服务提供方 ##

Eureka服务提供方(集群)

- microservicecloud-provider-dept-8001
- microservicecloud-provider-dept-8002
- microservicecloud-provider-dept-8003


带有hystrix功能的服务提供方(hystrix Demo)：

- microservicecloud-provider-dept-hystrix-8001

## 2.3服务消费方 ##

使用restTemplate+ribbon的方式来测试(ribbon自定义了负载均衡算法)

- microservicecloud-consumer-dept-80


使用feign来调用远程服务

- microservicecloud-consumer-dept-feign(直接调用microservicecloud-api的service接口)


监控消费方的指标(性能)

- microservicecloud-consumer-hystrix-dashboard


使用方式：

- 启动consumer-hystrix-dashboard项目，打开`http://localhost:9002/hystrix.stream`


![](https://i.imgur.com/yoDFMHg.png)

我们可以监控：`microservicecloud-provider-dept-hystrix-8001`这个项目，于是在输入栏输入`http://localhost:8001/actuator/hystrix.stream`

![](https://i.imgur.com/pBhQJkD.png)

随后，我们去测试接口：`http://localhost:8001/dept/get/7`


监控的数据就会变化了：

![](https://i.imgur.com/ITs9WPS.png)



## 2.4网关 ##


用于转发路由，服务过滤(安全验证)，限流等等：

- microservicecloud-zuul-gateway-9527

## 2.5Cloud配置文件 ##


git脚本回顾：

```git

1. git init //初始化仓库

2. git add .(文件name) //添加文件到本地仓库

3. git commit -m "first commit" //添加文件描述信息

4. git remote add origin + 远程仓库地址 //链接远程仓库，创建主分支

5. git pull origin master --allow-unrelated-histories  // 把本地仓库的变化连接到远程仓库主分支

6. git push -u origin master //把本地仓库的文件推送到远程仓库


```

SpringCloud Config服务端(获取配置都从这里来拿)

- microservicecloud-config-3344


SpringCloud Config 客户端：

- microservicecloud-config-client-3355
- microservicecloud-config-dept-client-8001
- microservicecloud-config-eureka-client-7001




## 2.6tip ##

如果使用的是IDEA的话，使用 Run dashboard比较方便~

![](https://i.imgur.com/CgrUIwL.png)

# 最后 #

获取更多**原创文章**，公众号：Java3y


