---
title: 微服务之配置服务器切换profile
date: 2017-11-16
categories: Ops
tags:
- Ops
- Spring Boot
- Docker
- Maven
---
最近遇到Spring-boot的多个profile切换问题，需求是这样的：微服务中引入了Spring Cloud Config，服务启动的时候，从Config Server中读取该实例对应的配置信息。本地开发环境可能使用的profile是default，到了集成测试环境就需要切换到jenkins，到了预发布环境又变成了prod。多个profile需要之间可以切换。

这边设置的时候还走了点弯路，先是探索了一遍pom的profile，后来才到Spring-boot的配置文件。

这两部分实现的功能不太一样，本文将会具体讲下这两部分。

## 1. profile之Maven
 maven切换profile的命令很简单，加上`-P`参数指定你的profile，如指定prod：
 
 ```bash
 mvn clean package -P prod
 ```
 maven使用名字为prod的profile来打包，即所有的配置文件都使用生产环境。   
 下面看下pom中的profiles：
 
 ```xml
 <profiles>
   <profile>
      <id>dev</id>
      <activation>
         <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
         <profileActive>dev</profileActive>
      </properties>
      <dependencies>
         <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
         </dependency>
      </dependencies>
   </profile>

   <profile>
      <id>prod</id>
      <dependencies>
         <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
         </dependency>
      </dependencies>
      <properties>
         <profileActive>prod</profileActive>
      </properties>
   </profile>
</profiles>
 ```
 
 对于resources的配置如下：
 
 ```xml
 
 	<build>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <!-- 过滤掉所有配置文件-->
                <excludes>
                    <exclude>application-dev.yml</exclude>
                    <exclude>application-prod.yml</exclude>
                </excludes>
            </resource>
            <resource>
                <filtering>true</filtering>
                <directory>src/main/resources</directory>
                <!--根据profile中的变量profileActive指定对应的配置文件-->
                <includes>
                    <include>application-${profileActive}.yml</include>
                </includes>
            </resource>
        </resources>
    </build>
 ```
 
 上面的两段pom配置相结合，当指定profile为prod时，环境变量profileActive的属性值变为prod。指定打包时，包含application-prod.yml。
 
 所以当你有多套配置文件，可以动态根据mvn命令的参数-P动态指定你所需要加载的配置文件。
 
## 2. profile之Spring boot

Profile是Spring boot用来针对不同环境对不同配置提供支持的,全局Profile配置使用。
application-{profile}.yml 如:application-yml。

spring通过配置spring.profiles.active指定激活某个具体的profile。除了spring.profiles.active来激活一个或者多个profile之外，还可以用spring.profiles.include来叠加profile。


```yml
spring.profiles.include: prod,dev
```
下面看一下我们的application.yml中包含的配置：

```yml

spring:
  profiles:
    active: dev
---
#开发环境配置
spring:
  profiles: dev
server:
  port: 8080
---
#测试环境配置
spring:
  profiles: test
server:
  port: 8081
---
#生产环境配置
spring:
  profiles: prod

server:
  port: 8082
```
application.yml文件分为四部分，使用一组(---)来作为分隔符。第一部分，通用配置部分，表示三个环境都通用的属性，默认激活了dev的profile；后面三部分分别表示不同的环境，指定了不同的port。

部署到服务器的话，正常会打成jar包，加上参数
`--spring.profiles.active=test`指定加载哪个环境的配置。

在IDE中也可以直接配置激活的profile。

![entry](http://ovcjgn2x0.bkt.clouddn.com/profile%E5%85%A5%E5%8F%A3.jpg "idea配置")

![profile](http://ovcjgn2x0.bkt.clouddn.com/configprofiles.jpg "idea配置profile")

## 3. config server的配置
这节讲下与Spring cloud config的结合使用。既然使用了config server，动态配置这块基本就由配置服务器完成了。配置服务器中对该服务指定多个profile。config Server中的配置优先于本地配置，当服务启动时，根据激活的profile，去配置服务器拉取其对应的配置。

既然知道了上面的主要流程，就可以明白我们的需求其实是要在服务启动时指定激活的profile。所以上面一节关于Spring boot的profile动态配置，我们的问题就能解决了。但上面讲到的是jar包启动时指定`--spring.profiles.active`，实际都是微服务的容器化部署，服务通过容器直接启动jar包，这样就需要容器启动的时候能够动态指定active profile，所以上面的配置改一下，如下：

```yml
spring:
  profiles:
    active: ${ACTIVE_PROFILE:dev}
```
![startup](http://ovcjgn2x0.bkt.clouddn.com/configstartup.jpg "容器启动截图")

从容器的启动截图来看，指定了`docker run -d -e  ACTIVE_PROFILE=exp ...`后，active profile 变味了exp，并且从config server中拉取对应的是gatewayserver的exp配置。

## 4. 总结
本文主要写了Spring-boot配置服务器切换profile。首先描述了需求背景，然后是对maven pom中profile进行了探索与讲解，其次是讲解了Spring-boot中的profile切换，最后结合config server实现容器部署微服务的profile。笔者最开始一直认为通过pom的profile切换就可以设置服务启动的profile，经过一番探索，发现与配置服务器结合好像并不需要pom的profile这么繁琐，结合配置服务器可以更方便的使用Spring boot的profile。


---
### 参考

1. [详解Spring Boot Profiles 配置和使用](http://www.jb51.net/article/115673.htm)
2. [[Spring Boot 系列] 集成maven和Spring boot的profile功能](http://blog.csdn.net/lihe2008125/article/details/50443491)
