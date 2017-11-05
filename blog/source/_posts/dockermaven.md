---
title: 微服务部署之Maven插件构建Docker镜像
date: 2017-11-02
categories: Ops
tags:
- Docker
- Ops
-  Spring Cloud
---
## 1.背景
微服务架构下，微服务在带来良好的设计和架构理念的同时，也带来了运维上的额外复杂性，尤其是在服务部署和服务监控上。单体应用是集中式的，就一个单体跑在一起，部署和管理的时候非常简单，而微服务是一个网状分布的，有很多服务需要维护和管理，对它进行部署和维护的时候则比较复杂。  
 
下面从Dev的角度来看一下Ops的工作。从Dev提交代码，到完成集成测试的一系列步骤如下：

- 首先是开发人员把程序代码更新后上传到Git，然后其他的事情都将由Jenkins自动完成。
- 然后Git在接收到用户更新的代码后，会把消息和任务传递给Jenkins，然后Jenkins会自动构建一个任务，下载Maven相关的软件包。下载完成后，就开始利用Maven Build新的项目包，然后重建Maven容器，构建新的Image并Push到Docker私有库中。
- 最后删除正在运行的Docker容器，再基于新的镜像重新把Docker容器启动，自动完成集成测试。

整个过程都是自动的，这样就简化了原本复杂的集成工作，一天可以集成一次，甚至是多次。


![Ops](http://img.blog.csdn.net/20160830130543156 "DevOps流程图")

本文主要关注的第二步，作为Dev使用Maven插件构建Docker镜像。

## 2. 过程步骤
### 2.1 环境

笔者的电脑系统是MacOS，在进行下面的步骤之前，先具备一下条件：

- Docker Registry
- Maven（3.5.0）
- JDK(1.8.0_131)
- Docker for Mac (17.09.0-ce-mac35)

Maven 和JDK 就不用过多多了，必须具有的。Docker Registry是私有的hub，mac上装好docker之后，配置一下Docker Registry的地址，配置如下：

```json
{
  "debug" : true,
  "experimental" : false,
  "insecure-registries" : [
    "192.168.1.202"
  ]
}
```

![config](http://ovcjgn2x0.bkt.clouddn.com/docker%20%E9%85%8D%E7%BD%AE.jpg "docker 配置")

### 2.2 pom文件
pom文件中需要引入相应的插件。docker-maven-plugin有三款：spotify、fabric8io和bibryam。其中第一款最为流行，资料也多，所以毫不犹豫选择第一款。
插件有两种使用方式，一种是在直接在pom配置中指定baseImage和entryPoint。另一种适合于复杂的构建，使用dockerfile，只需要在配置中指定dockerfile的位置。前一种比较简单，此处略过，主要讲下第二种的配置。

```xml
			<plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${maven.docker.version}</version>
                <!--插件绑定到phase-->
                <executions>
                    <execution>
                        <phase>install</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                <!--配置变量，包括是否build、imageName、imageTag，非常灵活-->
                    <skipDocker>${docker.skip.build}</skipDocker>
                    <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                    <!--最后镜像产生了两个tag，版本和和最新的-->
                    <imageTags>
                        <imageTag>${project.version}</imageTag>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <forceTags>true</forceTags>                 
                    <env>
                        <TZ>Asia/Shanghai</TZ>
                    </env>
                    <!--时区配置-->
                    <runs>
                        <run>ln -snf /usr/share/zoneinfo/$TZ /etc/localtime</run>
                        <run>echo $TZ > /etc/timezone</run>                      
                    </runs>
                    <dockerDirectory>${project.basedir}</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                    <!--push到私有的hub-->
                    <serverId>docker-registry</serverId>
                </configuration>
            </plugin>	
```

${maven.docker.version}、${docker.skip.build}、${docker.image.prefix}都是可配置的变量。${project.basedir}、${project.build.directory}、${project.build.finalName}、${project.version}分别对应项目根目录、构建目录、打包后生成的结果名称、项目版本号。    
上面的pom插件配置，指定了dockerfile的位置和镜像的命名规则。并将docker的build目标，绑定在install这个phase上。

### 2.3 dockerfile

```bash
FROM 192.168.1.202/library/basejava
VOLUME /tmp
ADD ./target/cloud-api-gateway-1.2.0.RELEASE.jar app.jar
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```
  
dockerfile 写的很简单，将jar包ADD进去，提供ENTRYPOINT。

### 2.4 setting.xml
在pom插件中，还有一个serverId的配置。这个配置是必要的，对于需要将image上传到私有hub上，在如上配置之后，只需要加上`-DpushImage`即可实现。serverId是与maven的配置文件setting.xml相对应，setting.xml中增加的配置：

```xml
  <server>
    <id>docker-registry</id>
    <username>用户名</username>
    <password>密码</password>
    <configuration>
      <email>邮箱</email>
    </configuration>
  </server>
```

### 2.5 结果

![res](http://ovcjgn2x0.bkt.clouddn.com/resultdocker.jpg "执行结果")

上图是执行`mvn clean install -DpushImage`成功的结果。mvn首先是打包，将生产的文件拷贝到target下的docker目录，然后执行dockerfile中的步骤，将打成的镜像进行tag，最后上传到私有hub上。

![har](http://ovcjgn2x0.bkt.clouddn.com/harbor.jpg "成功上传镜像")

上图是VMware Harbor中的截图，可以看到，我们已经成功将镜像上传，其tag有两个：1.2.0.RELEASE和latest。

## 3. 总结
本文属于工程实践类文章，比较简单。开头由背景介绍了Dev到继承测试的一系列步骤，本文主要关注的是第二步，作为Dev使用Maven插件构建Docker镜像。正文部分主要讲了实践的环境，然后讲了docker-maven-plugin插件的使用方式，重点介绍了使用dockerfile的方式，对于涉及到的配置进行了解释。

**本文的源码地址：   
GitHub：https://github.com/keets2012/snowflake-id-generator      
码云： https://gitee.com/keets/snowflake-id-generator**

---

### 参考
1. [从运维的角度看微服务和容器](http://geek.csdn.net/news/detail/98271)
2. [Docker与微服务-使用Maven插件构建Docker镜像](http://blog.csdn.net/qq_22841811/article/details/67369530#reply)


