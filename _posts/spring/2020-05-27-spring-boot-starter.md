---
layout: post
title: 如何自定义一个 SpringBoot Starter
categories: [Spring]
description: 如何自定义一个 SpringBoot Starter
keywords: spring, springboot, starter
---

## 1 前言
从前从前，有个面试官问我一个 `SpringBoot Starter` 的开发流程，我说我没有写过 starter，然后就没有然后了，面试官说我技术深度不够。

我想说这东西不是很简单吗，如果要自己写一个出来也是分分钟的事情。至于就因为我没有写过 starter 就觉得我一点都不会 SpringBoot 吗？

当然我当时确实水平不足，连 Java 的 SPI 都忘了是啥，后来又捡了起来，原来我在大学的时候就用过 Java 的 SPI，悔之晚矣！如果你也不知道什么是 SPI，可以去看看的我另一篇文章[JVM 双亲委派模型及 SPI 实现原理分析]。

## 2 什么是 SpringBoot starter
starter 是 SpringBoot 的一个重要的组成部分，它相当于一个集成的模块，比如你想用 Mybatis 和 lombok，但是在 pom 文件中需要写两个依赖，如果你将他们集成为一个 starter（或者将更多你需要的依赖集成进去），那么你只需要在 pom 文件中写一个 starter 依赖就可以了，这对于一个可复用模块的开发和维护都极为有利。

同时，在 maven 中引入 starter 依赖之后，SpringBoot 就能自动扫描到要加载的信息并启动相应的默认配置，它遵循“约定大于配置”的理念。

## 3 如何开发一个 SpringBoot starter
### 3.0 环境说明
- jdk 1.8.0_151
- maven 3.6.3
- IDEA 编译器
- 要开发的 starter 是一个日期格式化工具，它的功能是可以指定如何格式转化日期类型，同时在配置文件中可以开启关闭此功能

### 3.1 创建项目
在 IDEA 中新建一个 maven 工程，如下图所示。

![1.png](https://planeswalker23.github.io/images/posts/2020-05-27-1.png)

> Spring 官方建议自定义的 starter 使用 `xxx-spring-boot-starter` 命名规则，以区分 SpringBoot 生态提供的 starter。

然后需要在 pom 文件中加入实现 starter 所需要的依赖。

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.walker.planes</groupId>
    <artifactId>date-format-spring-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.3.0.RELEASE</version>
        </dependency>
    </dependencies>
</project>
```

### 3.2 创建配置信息实体类
在开发 SpringBoot 项目的时候，我们有时候会在 `application.properties` 配置文件中进行一些配置的修改，用以开启或修改相关的配置，如 `server.port` 可以用来修改项目启动的端口，所以在编写自定义日期格式化 starter 时，也可以用到这个功能，根据配置文件来开启或关闭某项功能。

我们在 `application.properties` 配置文件中新增两个配置项，第一个是用来控制该 starter 的启动或关闭，第二个是需要指定的格式化信息。

然后创建一个配置文件映射类，它的私有成员变量 pattern 就是需要在配置文件中指定的属性，如果没有指定默认值为 `yyyy-MM-dd HH:mm:ss`。

```java
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 配置信息实体类
 * @author Planeswalker23
 * @date 2020-05-25
 */
@ConfigurationProperties("formatter")
public class DateFormatProperties {

    /**
     * default format pattern
     */
    private String pattern = "yyyy-MM-dd HH:mm:ss";

    // 忽略 getter setter
}
```

`@ConfigurationProperties(prefix = "formatter")` 注解的作用是将相同前缀的配置信息通过配置项名称映射成实体类。在这里就可以将 `application.properties` 配置文件中前缀是 formatter 的配置项映射到 DateFormatProperties 类中。

### 3.3 创建配置类
然后我们需要创建一个配置类，这个配置类能够将核心功能类注入到 Ioc 容器，使得在其他引用此 starter 的项目中能够使用自动配置的功能。

```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.text.SimpleDateFormat;
/**
 * 配置信息实体类
 * @author Planeswalker23
 * @date 2020-05-25
 */
@Configuration
@EnableConfigurationProperties(DateFormatProperties.class)
@ConditionalOnProperty(prefix = "formatter", name = "enabled", havingValue = "true")
public class DateFormatConfiguration {
    private DateFormatProperties dateFormatProperties;
    public DateFormatConfiguration(DateFormatProperties dateFormatProperties) {
        this.dateFormatProperties = dateFormatProperties;
    }

    @Bean(name = "myDateFormatter")
    public SimpleDateFormat myDateFormatter() {
        System.out.println("start to initialize SimpleDateFormat with pattern: " + dateFormatProperties.getPattern());
        return new SimpleDateFormat(dateFormatProperties.getPattern());
    }
}
```

- 这里 starter 的功能是基于 SimpleDateFormat 类的，其实就是解析配置文件中的指定格式，将其设置为 SimpleDateFormat 的 pattern 属性，然后将 SimpleDateFormat 类作为“组件”注册在 Ioc 容器中，这样我们就可以在引用了该 starter 的项目中使用自定义格式化功能了。
- `@Configuration` 注解的作用是将 DateFormatConfiguration 类作为配置类注入容器
- `@EnableConfigurationProperties` 注解的作用是开启资源实体类的加载，也就是说开启配置文件映射为资源实体类的功能。
- `@ConditionalOnProperty` 注解是开启条件化配置，也就是说只有在配置文件中的 formatter.enabled 属性的值为 true 时，这个 starter 才会生效。

### 3.4 指定自动装配
至此开发的业务代码就结束了，但是还有一个最重要的步骤就是指定 DateFormatConfiguration 类作为自动装配类。

这需要在 resource 目录下创建 MATE-INF 文件夹，并创建一个名为 spring.factories 的文件，然后在文件中写下

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.walker.planes.DateFormatConfiguration
```

这行代码的意思就是将 DateFormatConfiguration 设置为自动装配类，在 SpringBoot 工程启动时会去扫描 MATE-INF 文件夹的 spring.factories 文件，并加载这个文件中指定的类，启动自动装配功能。

### 3.5 发布&测试
然后在命令行中敲下 `mvn clean install` 命令，当看到下面的输出后，自定义的 starter 就可以作为一个依赖被引用了。

```java
[INFO] -----------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] -----------------------------------------------------------
[INFO] Total time:  1.470 s
[INFO] Finished at: 2020-05-25T20:24:49+08:00
[INFO] -----------------------------------------------------------
```

新建一个测试工程，在 pom 文件中加入如下的依赖。

```java
<dependency>
    <groupId>org.walker.planes</groupId>
    <artifactId>date-format-spring-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

然后在 application.properties 文件中开启自定义 starter 的配置。

```java
formatter.enabled=true
formatter.pattern=yyyy-MM-dd
```

然后在启动类中创建一个 Runner 任务，简单的输出一个日期类，就像下面这样。

```java
@SpringBootApplication
public class FirstStarterApplication implements ApplicationRunner {

	public static void main(String[] args) {
		SpringApplication.run(FirstStarterApplication.class, args);
	}

	@Override
	public void run(ApplicationArguments args) throws Exception {
		System.out.println(simpleDateFormat.format(new Date()));
	}

	@Resource(type = SimpleDateFormat.class)
	private SimpleDateFormat simpleDateFormat;
}
```

启动项目，我们可以看到自动配置生效了，程序将配置文件中指定的格式化样式覆盖了默认的 pattern。

```java
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.0.RELEASE)

2020-05-25 20:31:27.144  INFO 5559 --- [           main] org.test.FirstStarterApplication         : Starting FirstStarterApplication on fandeMac-mini.local with PID 5559 (/Users/fan/workspace/first-starter/target/classes started by fan in /Users/fan/workspace/first-starter)
2020-05-25 20:31:27.146  INFO 5559 --- [           main] org.test.FirstStarterApplication         : No active profile set, falling back to default profiles: default
start to initialize SimpleDateFormat with pattern: yyyy-MM-dd
2020-05-25 20:31:27.792  INFO 5559 --- [           main] org.test.FirstStarterApplication         : Started FirstStarterApplication in 0.894 seconds (JVM running for 1.4)
2020-05-25
```

## 小结
至此，我们已经完成了一个简单的 SpringBoot Start 的开发了，最后来总结一下一个 starter 的开发流程。
1. 引入 `spring-boot-autoconfigure` 依赖
2. 创建配置实体类
3. 创建自动配置类，设置实例化条件（@Conditionalxxx注解，可不设置），并注入容器
4. 在 MATE-INF 文件夹下创建 `spring.factories` 文件夹，激活自动配置。
5. 在 maven 仓库发布 starter

