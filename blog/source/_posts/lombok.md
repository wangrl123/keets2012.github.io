---
title: Lombok使用与原理
date: 2017-10-6 
categories: Utils
tags:
- Lombok
---
## 1. Lombok简介
首先`Lombok`是一款Java IDE的应用工具插件，一个可以通过简单的注解形式来帮助我们简化消除一些必须有但显得很臃肿的Java代码的工具，比如属性的构造器、getter、setter、equals、hashcode、toString方法。结合IDE，通过使用对应的注解，可以在编译源码的时候生成对应的方法。官方地址：`https://projectlombok.org/`。    

虽然上述的那些常用方法IDE都能生成，但是lombok更加简洁与方便，能够达到的效果就是在源码中不需要写一些通用的方法，但是在编译生成的字节码文件中会帮我们生成这些方法，这就是lombok的神奇作用。

## 2. 安装
### 2.1 插件安装
笔者主要使用的IDE是Intellij idea，编译器需要在

```
preference->plugins->Browse repositories
```

搜索lombok，然后安装plugins，需要稍等片刻。笔者截图已经安装好。
![lombok](http://ovcjgn2x0.bkt.clouddn.com/lombok.png "lombok插件")

### 2.2 添加jar包
在项目中添加lombok的jar包，笔者用的是maven，所以在pom文件中添加了如下的依赖。gradle使用见官网。

```pom
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.16</version>
            <scope>provided</scope>
        </dependency>
```

## 3. 使用
lombok主要通过注解起作用，详细的注解见[Lombok features](https://projectlombok.org/features/all)。   

With Lombok:   

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@AllArgsConstructor
public class UserEntity implements Serializable {
    private long userId;
    private String userName;
    private String sex;
}
```

编译后：   

```java

import java.beans.ConstructorProperties;
import java.io.Serializable;

public class UserEntity implements Serializable {
    private long userId;
    private String userName;
    private String sex;

    public long getUserId() {
        return this.userId;
    }

    public String getUserName() {
        return this.userName;
    }

    public String getSex() {
        return this.sex;
    }

    public void setUserId(long userId) {
        this.userId = userId;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public boolean equals(Object o) {
        if(o == this) {
            return true;
        } else if(!(o instanceof UserEntity)) {
            return false;
        } else {
            UserEntity other = (UserEntity)o;
            if(!other.canEqual(this)) {
                return false;
            } else if(this.getUserId() != other.getUserId()) {
                return false;
            } else {
                Object this$userName = this.getUserName();
                Object other$userName = other.getUserName();
                if(this$userName == null) {
                    if(other$userName != null) {
                        return false;
                    }
                } else if(!this$userName.equals(other$userName)) {
                    return false;
                }

                Object this$sex = this.getSex();
                Object other$sex = other.getSex();
                if(this$sex == null) {
                    if(other$sex != null) {
                        return false;
                    }
                } else if(!this$sex.equals(other$sex)) {
                    return false;
                }

                return true;
            }
        }
    }

    protected boolean canEqual(Object other) {
        return other instanceof UserEntity;
    }

    public int hashCode() {
        int PRIME = true;
        int result = 1;
        long $userId = this.getUserId();
        int result = result * 59 + (int)($userId >>> 32 ^ $userId);
        Object $userName = this.getUserName();
        result = result * 59 + ($userName == null?43:$userName.hashCode());
        Object $sex = this.getSex();
        result = result * 59 + ($sex == null?43:$sex.hashCode());
        return result;
    }

    public String toString() {
        return "UserEntity(userId=" + this.getUserId() + ", userName=" + this.getUserName() + ", sex=" + this.getSex() + ")";
    }

    @ConstructorProperties({"userId", "userName", "sex"})
    public UserEntity(long userId, String userName, String sex) {
        this.userId = userId;
        this.userName = userName;
        this.sex = sex;
    }
}
```

这边介绍笔者经常使用到的注解。

- val，用在局部变量前面，相当于将变量声明为final

- @Value   
用在类上，是@Data的不可变形式，相当于为属性添加final声明，只提供getter方法，而不提供setter方法

- @Data   
@ToString, @EqualsAndHashCode, 所有属性的@Getter, 所有non-final属性的@Setter和@RequiredArgsConstructor的组合，通常情况下，我们使用这个注解就足够了。


- @NoArgsConstructor无参构造器

- @AllArgsConstructor全参构造器

- @ToString   
 生成toString方法，默认情况下，会输出类名、所有属性，属性会按照顺序输出，以逗号分割。
 
- @EqualsAndHashCode   
 默认情况下，会使用所有非瞬态(non-transient)和非静态(non-static)字段来生成equals和hascode方法，也可以指定具体使用哪些属性。

- @Getter / @Setter   
上面已经说过，一般用@data就不用额外加这个注解了。可以作用在类上和属性上，放在类上，会对所有的非静态(non-static)属性生成Getter/Setter方法，放在属性上，会对该属性生成Getter/Setter方法。并可以指定Getter/Setter方法的访问级别。

- @NonNull，给方法参数增加这个注解会自动在方法内对该参数进行是否为空的校验，如果为空，则抛出NPE（NullPointerException）

- @Cleanup   
自动管理资源，用在局部变量之前，在当前变量范围内即将执行完毕退出之前会自动清理资源，自动生成try-finally这样的代码来关闭流

- @Log   
根据不同的注解生成不同类型的log对象，但是实例名称都是log，有7种可选实现类：   

	1). @Log4j
	
	```java
	private static final org.apache.log4j.Logger log = org.apache.log4j.Logger.getLogger(LogExample.class);
	```
	2). @Log4j2
			
	```java
	 private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.LogManager.getLogger(LogExample.class);
	```
	3). @Slf4j
	
	```java 
	 private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExample.class);
	```
	4). @XSlf4j

	```java 
	private static final org.slf4j.ext.XLogger log = org.slf4j.ext.XLoggerFactory.getXLogger(LogExample.class);
	```
	5). @CommonsLog
	
	```java 
	private static final org.apache.commons.logging.Log log = org.apache.commons.logging.LogFactory.getLog(LogExample.class);
	```
	6). @JBossLog
	
	```java 
	private static final org.jboss.logging.Logger log = org.jboss.logging.Logger.getLogger(LogExample.class);
	```
	7). @Log
	
	```java
	private static final java.util.logging.Logger log = java.util.logging.Logger.getLogger(LogExample.class.getName());
	``` 	  
	默认情况下，logger的名字将会是被@Log注解的那个类的名字。当然这也可以被个性化命名，通过topic参数，如`@XSlf4j(topic="reporting")`。

## 4. 原理
lombok 主要通过注解生效，自jdk5引入注解，由两种解析方式。   
第一种是运行时解析，`@Retention(RetentionPolicy.RUNTIME)`, 定义注解的保留策略，这样可以通过反射拿到该注解。   
另一种是编译时解析，有两种机制。

- Annotation Processing Tool，apt自JDK5产生，JDK7已标记为过期，不推荐使用，JDK8中已彻底删除，自JDK6开始，可以使用Pluggable Annotation Processing API来替换它，apt被替换主要有2点原因。api都在com.sun.mirror非标准包下，还有就是没有集成到javac中，需要额外运行。

- Pluggable Annotation Processing API   
lombok使用这种方式实现，基于JSR 269，自JDK6加入，作为apt的替代方案，它解决了apt的两个问题，javac在执行的时候会调用实现了该API的程序，这样我们就可以对编译器做一些增强，这时javac执行的过程如下：

![pap](http://ovcjgn2x0.bkt.clouddn.com/pap.png "javac执行流程")

## 5. 总结
这篇文章主要讲解了lombok的入门与使用。介绍了一些常用的lombok注解，大大简化了我们的开发工作和代码的简洁性。当然，lombok不支持多种参数构造器的重载，工具毕竟是工具，我感觉并不会有非常完美适合每个人的工具。最后，我个人还是很推荐这款插件的，毕竟我很懒，😆。


----
### 参考
1. [Lombok Docs](https://projectlombok.org/features/all)
2. [Java奇淫巧技之Lombok](http://blog.csdn.net/ghsau/article/details/52334762)

