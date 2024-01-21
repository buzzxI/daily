# 前置工作

这里的项目新建成springboot的父子项目的格式，方便项目管理

新建一个`maven`的项目，只保留一个`pom.xml`文件。

在`pom.xml`中添加：`<packaging>pom</packaging>`，打包的时候就不会打成jar包了

在`pom.xml`中进行依赖管理的时候，使用标签`<dependencyManagement>`。在这个标签内部的依赖并不会直接被子项目继承，而需要子项目实际引入。子项目引入的时候不指定版本号，默认会继承父项目的版本号，而如果指定了，就以子项目的版本号为准。

其和单独的`<dependencies>`最大的区别在于，父项目中的`<dependencies>`依赖会被子项目全部继承。所以如果使用了`<dependencyManagement>`，一定要在子项目里声明依赖。

引入版本的时候参考：[版本说明 · alibaba/spring-cloud-alibaba Wiki (github.com)](https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明)

父项目的`pom.xml`文件如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.buzz</groupId>
    <artifactId>tough_springcloud</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!--springboot 依赖-->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.6.3</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--spring cloud alibaba依赖-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2021.0.1.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--druid 依赖-->
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>druid-spring-boot-starter</artifactId>
                <version>1.2.8</version>
            </dependency>
            <!--mybatis 依赖-->
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>2.2.2</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

## 构建微服务模块的一般操作

* 新建module：建好了之后在父项目中多了`<modules>`标签，里面就是我们新建的module的项目名

* 修改子项目pom.xml：其实没啥的，就是添加依赖，注意不需要写版本号了

* 编写子项目yaml文件：在resource目录下新建`application.yaml`文件表示配置文件，必须是这个名字，这样springboot可以正常自动识别

* 子项目的主启动类：就是一个典型的springboot的项目的写法（这里以nacos微服务提供商举例）：

  ```java
  package com.buzz;
  
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.boot.SpringApplication;
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  
  /**
   * @program: tough_springcloud
   * @description:
   * @author: buzz
   * @create: 2022-05-03 20:44
   **/
  @SpringBootApplication
  @EnableDiscoveryClient
  public class NacosProviderMain {
  	public static void main(String[] args) {
  		SpringApplication.run(NacosProviderMain.class, args);
  	}
  }
  
  ```

* 业务逻辑...

# OpenFeign

[Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)

主要是进行服务调用的，使用接口加注解，就可以是实现远程调用。

考虑一个消费者和服务提供者的场景，消费者需要调用服务提供者暴露的服务，其实可以将其抽象为接口中的方法，进行方法的调用。在消费者端也定义`Service`接口，并将其声明为`Feign`类型，从而消费者通过调用接口中的方法，实现远程调用。

`OpenFeign`集成了`Ribbon`，可以实现负载均衡。

## 简单的使用

* 在`pom.xml`中添加`openfeign`的依赖

  使用的时候，需要在消费者端添加依赖：

  ```xml
  <!--openfeign依赖-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

* 在主启动类上添加注解`@EnableFeignClients`，表示激活`OpenFeign`

* 在消费者的子项目中添加`Service`接口，并在接口上添加注解`@FeignClient`

  > 这里不需要添加`@Service`

  因为`OpenFeign`支持`SpringMVC`注解，所以可以使用类似：`@RequestMapping`这种，直接调用其方法

* 注解`@FeignClient`中明确调用的服务提供者的名称

## fallback

这部分主要看[Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#spring-cloud-feign-circuitbreaker-fallback)

注意使用`fallback`的时候，需要和断路器一起使用，这里因为使用了`sentinel`作为流控熔断，所以这里需要添加`sentinel`依赖，在`yaml`文件中添加：

```yaml
feign:
  sentinel:
	enable: true
```

如果远程调用方法出现了错误，可以通过设置fallback对应的类。

简单来说比如定义了接口：

```java
package com.buzz.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-07 20:09
 **/
@FeignClient(value = "${provider-name}", fallback = NacosProviderServiceFallBackImpl.class)
public interface NacosProviderService {

	@RequestMapping("/hello/{msg}")
	String helloNacos(@PathVariable(name = "msg") String msg);
}
```

定义的`fallback`的类其实就是这个接口的一个实现类

```java
package com.buzz.service;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-07 21:46
 **/
public class NacosProviderServiceFallBackImpl implements NacosProviderService{
	@Override
	public String helloNacos(String msg) {
		return "方法调用失败";
	}
}
```

如果调用失败了，就会调用本地的方法，表示调用失败，

如果仅设置了一个`fallback`属性只能知道调用失败了，却不能知道失败的具体信息，比如错误码之类的，这个时候就需要设置属性`fallbackFactory`

```java
package com.buzz.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-07 21:54
 **/
@Component
// 注意这里类需要实现接口FallbackFactory，泛型就传入我们定义的FeignClient接口
public class NacosProviderFallBackFactory implements FallbackFactory<NacosProviderService> {
   private NacosProviderServiceFallBackImpl fallback;

   @Autowired
   public void setFallback(NacosProviderServiceFallBackImpl fallback) {
      this.fallback = fallback;
   }
	
    // 这个工厂的关键方法，它可以获取到异常，这个异常中包含了错误信息，
    // 要注意的是这个方法需要一个fallback类型的返回值
   @Override
   public NacosProviderService create(Throwable cause) {
      System.out.println("lalala");
      System.out.println(cause.getMessage());
      return fallback;
   }
}
```

## 超时调用

`OpenFeign`默认等待时间为`1s`，如果存在长业务的情况，需要手动设置超时时间

这里主要看[Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#timeout-handling)

这里的主要配置类为：

```java
@ConfigurationProperties("feign.client")
public class FeignClientProperties 
```

具体的配置参考了：[Setting Custom Feign Client Timeouts](https://www.baeldung.com/feign-timeout#per-client)

具体的定义如下：

```yaml
# 通过定义锚点，实现变量定义
&provider-name nacos-provider

feign:
  client:
    config:
      # 这里面的default就是全局的超时时间设置
      default:
        connect-timeout: 1000
        read-timeout: 1000
      # 这里为对不同服务的超时时间设置
      *provider-name:
        connect-timeout: 1000
        read-timeout: 2000
```

## 日志

这部分看[Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#feign-logging)

它一共具有四类日志级别：

* `NONE`：没有日志，默认就是这个
* `BASIC`：日志记录了请求方法、请求`URL`、响应状态码和请求时间
* `HEADERS`：在`BASIC`的基础上，日志记录了请求和响应头中的信息
* `FULL`：日志记录了请求和响应的全部信息：`header`、`body`、`metadata`

更为具体的配置如下：

```yaml
&provider-name nacos-provider

feign:
  client:
    config:
      default:
        connect-timeout: 1000
        read-timeout: 1000
        logger-level: full
      *provider-name:
        connect-timeout: 1000
        read-timeout: 2000
        logger-level: headers
```

结合了上面的请求超时和这里的日志级别

# Nacos

服务注册中心、服务配置中心

到`github`的`release`页面下载[Releases · alibaba/nacos (github.com)](https://github.com/alibaba/nacos/releases)

官方手册在：[什么是 Nacos](https://nacos.io/zh-cn/docs/what-is-nacos.html)

首先需要启动`nacos`，下载好的zip文件解压然后按照[Nacos 快速开始](https://nacos.io/zh-cn/docs/quick-start.html)

这里需要注意的是，它启动的时候需要添加参数`-m standalone`，表示单机模式启动，它这个默认是集群启动

然后就可以在本地的8848端口，进入后台了，用户名和密码都是nacos

如果要看文档的话，因为是通过`spring cloud`构建项目，所以需要看`spring cloud`的文档：[Spring Cloud Alibaba Reference Documentation (spring-cloud-alibaba-group.github.io)](https://spring-cloud-alibaba-group.github.io/github-pages/2021/en-us/index.htm)

一个简单的nacos的服务注册的案例，见[Nacos：Spring Cloud Alibaba服务注册与配置中心](http://m.biancheng.net/springcloud/nacos.html)

当然官网的例子也挺好：[Nacos discovery · alibaba/spring-cloud-alibaba Wiki ](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery)

## 服务注册中心

### 服务提供者

就是官网的示例，一点一点来的：[Spring Cloud Alibaba Reference Documentation (spring-cloud-alibaba-group.github.io)](https://spring-cloud-alibaba-group.github.io/github-pages/2021/en-us/index.html#_spring_cloud_alibaba_nacos_discovery)

首先需要新建一个子项目，具体的步骤和见[前面](#构建微服务模块的一般操作)。

在`yaml`配置文件中添加：

```yaml
server:
  port: 8081
spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      discovery:
        server-addr: buzzx.icu:8848
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

这里要注意的是，在主启动类上还需要加上注解：`@EnableDiscoveryClient`

```java
package com.buzz;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-03 20:44
 **/
@SpringBootApplication
@EnableDiscoveryClient
public class NacosProviderMain {
	public static void main(String[] args) {
		SpringApplication.run(NacosProviderMain.class, args);
	}
}
```

然后写一个测试的`controller`：

```java
package com.buzz.controller;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-03 21:03
 **/
@RestController
public class HelloController {

	@RequestMapping("/hello/{msg}")
	public String helloNacos(@PathVariable(name = "msg", required = false) String msg) {
		return "this is nacos provider " + msg;
	}
}
```

nacos本身支持负载均衡，所以可以启动多个同一个名称的服务提供者，这里建立了两个完全重名的服务提供者，端口分别为`8081`和`8082`

### 服务消费者

消费者通过`RestTemplate`，调用服务提供者的接口。

还是新建一个子项目，`port`为8083，配置如下：

```yaml
server:
  port: 8083
spring:
  application:
    name: nacos-consumer
  cloud:
    nacos:
      discovery:
        server-addr: buzzx.icu:8848
        
# 注意这里的url路径对应的是服务提供者暴露给nacos-server的spring.application.name
provider-addr: http://nacos-provider
```

然后是`controller`：

```java
package com.buzz.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-04 11:44
 **/
@RestController
public class RequestController {
	@Value("${provider-addr}")
	String providerUrl;
    // 这里需要一个RestTemplate，正常情况下都是通过bean的方式注入的
	private RestTemplate template;
	@Autowired
	public void setTemplate(RestTemplate template) {
		this.template = template;
	}

	@RequestMapping("/consumer/{msg}")
	public String consume(@PathVariable("msg") String msg) {
		String url = providerUrl + "/hello/" + msg;
		System.out.println(url);
		return template.getForObject(url, String.class);
	}
}
```

这里要注意的是对于`RestTemplate`的配置：

```java
package com.buzz.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-04 11:42
 **/
@Configuration
public class RestTemplateConfig {

	@Bean
    /* 
    	这个很重要，说是Ribbon的负载均衡
    	不过，就算只有一个服务提供者，如果不加这个注解也是用不了的
    */
	@LoadBalanced
	public RestTemplate restTemplate() {
		return new RestTemplate();
	}
}
```

### CAP模型

> Nacos支持AP和CP模型之间的切换

## 服务配置中心

将服务拆分为微服务，每个服务相对小，而所有的服务数量多。可能出现重复的服务的情况，比如上面的通过启用两个服务提供者，并借助nacos实现负载均衡。通过服务配置中心，可以统一的配置每个微服务的配置。

通过微服务客户端获取配置，在项目初始化的时候需要从nacos-server获取配置

这里还是需要新建项目，并引入依赖：

```xml
<!--导入这个依赖后项目启动后会先加载bootstrap.yaml-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

编写配置文件，我们需要让项目在启动的时候就从`nacos-server`中获取配置，所以这里新建了两个配置文件：`bootstrap.yaml`和`application.yaml`

也正是为了支持`bootstrap.yaml`才引入了`spring-cloud-starter-bootstrap`依赖

在`bootstrap.yaml`中添加：

```yaml
server:
  port: 8084
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
    # 注意这里的discovery并不是必须的，只有微服务的服务注册使用的是nacos才需要这么写
      discovery:
        # nacos-server作为服务注册中心的地址
        server-addr: buzzx.icu:8848
      config:
        # nacos-server作为配置中心的地址
        server-addr: buzzx.icu:8848
        # 配置文件以yaml结尾
        file-extension: yaml
```

而在`application.yaml`中：

```yaml
spring:
  profiles:
    active: dev
```

表明了，我们启用了`dev`的配置文件

然后在`nacos`的后台需要新建配置文件：注意`Data Id`这个选项，其名称写法见[Spring Cloud Alibaba Reference Documentation](https://spring-cloud-alibaba-group.github.io/github-pages/2021/en-us/index.html#_support_configurations_at_the_profile_level)

默认的`nacos`会加载：`Data Id`为`${spring.application.name}.${file-extension}`和`${spring.application.name}-${profile}.${file-extension}`的文件，而真正服务使用的环境是`spring.profiles.active`决定的

这里因为常用的是`yaml`格式的配置文件，所以`file-extension`为yaml

在`nacos-server`的配置文件中，简单的添加了：

```yaml
user:
  name: buzz
  age: 10
  sex: male
```

然后编写`controller`：

```java
package com.buzz.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-04 17:52
 **/
@RestController
// 动态刷新配置
@RefreshScope
public class NacosConfigController {
	@Value("${user.name}")
	private String name;
	@Value("${user.age}")
	private int age;
	@Value("${user.sex}")
	private String sex;

	@RequestMapping("/config")
	public String getConfig() {
		return "name is:" + name  + "age is:" + age + "sex is:" + sex;
	}
}
```

### 命名空间

在不同的命名空间中可以具有相同的`Group`和`Data Id`，使用命名空间可以实现配置文件的分离

默认的命名空间是`public`，默认的`Group`是`DEFAULT_GROUP`；如果什么都不管，直接新建一个配置，那么就属于`public`下的`DEFAULT_GROUP`组下的配置文件

在`nacos-server`端，新建`namespace`和新建`Group`都是图形化的，非常友好

要注意的是，为了使用指定的命名空间中的配置，需要修改原来的`yaml`文件：

```yaml
server:
  port: 8084
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      discovery:
        # nacos-server作为服务注册中心的地址
        server-addr: buzzx.icu:8848
      config:
        # nacos-server作为配置中心的地址
        server-addr: buzzx.icu:8848
        # 配置文件以yaml结尾
        file-extension: yaml
        # 这里的namespace其实是自动生成的id
        namespace: f77d94eb-aa0c-4761-ba13-7243dad00061
        group: First_Group
```

## Nacos集群和持久化

`Nacos`本身内嵌了数据库，这也就是为什么重启`nacos-server`后，还可以在后台看到之前配置好的`yaml`文件。不过`nacos`可以使用本机自带的mysql数据库进行存储。具体的可以看[Nacos支持三种部署模式](https://nacos.io/zh-cn/docs/deployment.html)

对nacos进行集群部署，需要保证数据的一致性，所以最好使用外部数据源。

构建`Nacos`集群，具体见[集群部署说明 (nacos.io)](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html)

一个比较推荐的架构是：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/nacos-cluster.jpg)

大概的意思是，使用一个`SLB`做反向代理，这里使用的就是nginx。

因为只有一台服务器，这里使用的是单机模拟集群的方式，一台机器上，跑了三个nacos实例，每个占用不同的端口号（这意味着我们需要修改`appliacation.conf`将端口号修改）。这里的端口号为`8847`、`8848`、`8849`三个，我们需要将`nacos`复制三份，分别启动

> 为什么是三个，因为官网推荐集群最好是三个以上

主要就是配置好外部数据库，然后编写`cluster.conf`

> `cluster.conf`中最好使用`ifconfig`中的`ip`地址，`localhost`可能会有问题

这里的`cluster.conf`如下：

```properties
# 这个10.0.4.11是腾讯云的内网ip
10.0.4.11:8847
10.0.4.11:8848
10.0.4.11:8849
```

对于`nacos`集群，因为使用了`nginx`做反向代理，同时也实现了负载均衡。

这里并没有真正实现单机模拟三个集群，主要是因为确实服务器太差了，2G内存跑两个就卡死了。

在`nginx.conf`中加入：

```nginx
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    # 这里是重点，命名为nacos-cluster集群
    upstream nacos-cluster{
        server localhost:8847;
        server localhost:8848;
        server localhost:8849;
    }
    server {
        listen       8840;
        server_name  localhost;
        location / {
            # nginx代理的端口8840的请求转发到对应的集群中
            proxy_pass http://nacos-cluster;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

这里配置nacos集群的时候卡死了，优化的办法就是进入`startup.sh`中把`jvm`的启动选项修改了，比如堆区的大小，元空间的大小。不过腾讯云2G的确实还是不行，我都已经把堆区调成`32m`了，开两个还是会卡死，而这个`nacos`我也不知道他为什么对内存要求那么大，元空间这里最小都需要`128m`，不然直接`OOM`。

在`wsl`中重新部署了一下，发现如果同时开三个`nacos`，内存占用也会来到`50%`

还要注意的是，上面的端口号确实不太行，在`2.x`版本的`nacos`中，一个程序会占用4个端口号，具体的看：[单机搭建nacos集群2.+版本启动报端口被占用问题](https://blog.csdn.net/weixin_43646701/article/details/121783306)

所以最终的端口号：`8845`、`8848`、`8890`，`nginx`代理的端口号`8840`

# Sentinel

官方文档在[Spring Cloud Alibaba Reference Documentation](https://spring-cloud-alibaba-group.github.io/github-pages/2021/en-us/index.html#_spring_cloud_alibaba_sentinel)

> 只要把超链接中的`en-us`换成`cn-zh`就可以看到spring cloud的中文文档了
>
> 虽然感觉和机翻也没什么区别

上来就是一堆名词：服务降级、服务熔断、服务限流...

其实这个`Sentinel`也分为两部分，官网说是核心库和控制台，这个控制台就是图形化界面。

要运行`Sentinel`需要到github的release页面上下载[Releases · alibaba/Sentinel (github.com)](https://github.com/alibaba/Sentinel/releases)

下完就是一个jar包，直接运行就行。要注意的是，他默认占用了8080端口，正好和tomcat冲突，所以在运行之前需要看一下端口的占用情况。启动后，访问8080端口就可以看到`DashBoard`了。

> ```shell
> # 检查端口占用
> $ lsof -i:[port]
> ```

还是一样的，需要新建module，然后改pom，新建yaml，写启动类和controller

yaml文件长这样：

```yaml
spring:
  application:
    name: sentinel-config
  cloud:
    nacos:
      discovery:
        server-addr: buzzx.icu:8848
    sentinel:
      transport:
        # 这里填写启动了sentinel的地址
        dashboard: buzzx.icu:8080
        # 官网的解释是在8719端口开启一个http server，用来接受sentinel dashboard发送过来的指令
        # 比如说我们配置了限流，那么微服务将从8719端口收到sentinel dashboard发送过来的限流规则
        port: 8719
server:
  port: 8085
```

然后是controller：

```java
package com.buzz.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-06 15:03
 **/
@RestController
public class SentinelController {

	@RequestMapping("/config")
	public String hello() {
		return "this is sentinel";
	}
}
```

最开始，信心满满的，启动jar包，然后启动服务，然后sentinel一直报错。

现在解决的方式是：修改jdk版本，即便到了sentinel v1.84，对于高版本的jdk兼容还是很差；然后需要在yaml文件中添加一项：`clientIp`，不然`sentinel dashboard`找不到微服务

```yaml
spring:
  application:
    name: sentinel-config
  cloud:
    nacos:
      discovery:
        server-addr: buzzx.icu:8848
    sentinel:
      transport:
        # 这里填写启动了sentinel的地址，因为上云之后ip就是私网ip了
        # 就换成本机跑sentinel
        dashboard: localhost:8080
        # 官网的解释是在8719端口开启一个http server，用来接受sentinel dashboard发送过来的指令
        # 比如说我们配置了限流，那么微服务将从8719端口收到sentinel dashboard发送过来的限流规则
        port: 8719
        # 如果这个不写 sentinel 这个笨比真的找不到当前主机的位置
        clientIp: 127.0.0.1
server:
  port: 8085
```

## 流量控制

在图形化界面的解释：

* 资源名：默认请求路径。

* 针对来源：Sentinel可以针对调用者进行限流，填写微服务名，默认default（不区分来源）。

* 阈值类型/单机阈值：
  * QPS(每秒钟的请求数量)︰当调用该API的QPS达到阈值的时候，进行限流。
  * 线程数：当调用该API的线程数达到阈值的时候，进行限流。
  
  注意QPS和线程数的区别，线程数说的是服务**同时间可以并发处理的请求次数**。设置QPS为1，说明当前服务器只能在1s内接受1次请求；而设置线程数为1，说明服务器当前只能接受同时处理1个请求，此时QPS和服务器处理请求需要的时间有关，如果处理时间为0.5秒，那么理论上此时可以允许每秒请求2次。
  
* 流控模式：
    * 直接：API达到限流条件时，直接限流。
    
    * 关联：当关联的资源达到阈值时，就限流自己。
    
      此时设置流控时，会关联一个其他的资源，比如同时有两个资源：`/resource1`、`/resource2`，那么设置`/recource1`的流控模式为关联`/resource2`意味着，对`/resource2`请求超出了QPS（或线程数），那么此时将阻塞`/resource1`
    
      > 比如支付模块和订单模块，如果支付超出了限制，那么锁订单
    
    * 链路：只记录指定链路上的流量（指定资源从入口资源进来的流量，如果达到阈值，就进行限流)【API级别的针对来源】。
    
* 流控效果：

  如果流控模式选择线程数，那么默认就的流控效果就是快速失败。

  * 快速失败：直接失败，抛异常。
  * Warm up：根据Code Factor（冷加载因子，默认3）的值，从阈值/codeFactor，经过预热时长（默认单位为s），才达到设置的QPS阈值。
  * 排队等待：匀速排队，让请求以匀速的速度通过，阈值类型必须设置为QPS，否则无效。

一种比较简单的流量控制为：根据QPS，模式->直接，效果->快速失败。其效果为，当每秒钟调用超过一定次数后，新到的请求会被sentinel阻塞。

> 显然阻塞页面是可以自定义的

## 熔断降级

> 来自官网的概念：一个服务常常会调用别的模块，可能是另外的一个远程服务、数据库，或者第三方 API 等。例如，支付的时候，可能需要远程调用银联提供的 API；查询某个商品的价格，可能需要进行数据库查询。然而，这个被依赖服务的稳定性是不能保证的。如果依赖的服务出现了不稳定的情况，请求的响应时间变长，那么调用服务的方法的响应时间也会变长，线程会产生堆积，最终可能耗尽业务自身的线程池，服务本身也变得不可用。
>
> 因此我们需要对不稳定的**弱依赖服务调用**进行熔断降级，暂时切断不稳定调用，避免局部不稳定因素导致整体的雪崩。

熔断的策略：

- 慢调用比例 (`SLOW_REQUEST_RATIO`)：选择以慢调用比例作为阈值，需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。

  几个名词：RT（最大响应时间）、单位统计时长、最小请求数目、比例阈值、熔断时长

  * 最大响应时间（RT）：如果服务请求响应时间大于该设定值，那么认为出现了一次慢调用，在这里认为出现了异常
  * 单位统计时长：可以认为是一段时间，在一个时间窗口内会统计出现了慢调用的次数
  * 最小请求次数：如果在一个时间窗口内，出现了慢调用，但是整体请求次数没有达到最小请求次数，则不会触发熔断
  * 比例阈值：在满足了一个时间窗口内的请求次数大于最小请求次数，并且慢调用的占比大于比例阈值，会触发熔断
  * 熔断时长：在熔断时长的这部分时间内，远程服务将表示为不可用。

- 异常比例 (`ERROR_RATIO`)：当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。

  这个异常比例其实和上面的差不多，把上面的慢调用换成了这里的异常调用

- 异常数 (`ERROR_COUNT`)：当单位统计时长内的异常数目超过阈值之后会自动进行熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

  这个异常数和比例就没有关系了，他也不管系统是不是繁忙，只要单位时间内出现异常调用的次数超过了阈值，就熔断

## 热点参数限流

[热点参数限流 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/热点参数限流)

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

这里需要使用注解`@SentinelResource`定义资源，还是刚才的`controller`，现在如下：

```java
package com.buzz.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @program: tough_springcloud
 * @description:
 * @author: buzz
 * @create: 2022-05-06 15:03
 **/
@RestController
public class SentinelController {
	
    /**
     * 定义了一个名为hello的资源，
     * 如果服务被流控、降级、熔断，那么会调用默认的handler
     * 但是这里我们定义了一个handler，方法名为block，对应了下面的方法
	 */
	@SentinelResource(value = "hello", blockHandler = "block")
	@RequestMapping("/config/{name}/{age}")
	public String hello(
        @PathVariable(value = "name", required = false) String name,
		@PathVariable(value = "age", required = false) String age) {
		return "this is sentinel" + "name:" + name + " age:" + age;
	}

	/**
	 * @description 服务降级后调用的方法
	*/
	public String block(String name, String age, BlockException exception) {
		return "name:" + name + "age:" + age + "exception:" + exception.getMessage() + "blocked";
	}
}
```

现在认为热点参数为方法的第一个参数：`name`，

进入图形化界面，设置热点规则的参数索引和单机阈值，即可之间根据热点参数的流控

注意到上面自定义了`BlockHandler`，如果没有自定义且方法中没有声明`throws BlockException`，那么将抛出一个`UndeclaredThrowableException`异常

> 老版本的会返回一个字符串，1.8.4的就直接抛出异常了

在上述配置中是对方法中的参数进行限流，实际上，还可以根据参数取值进行限流。

具体的，在图形化界面中，通过参数例外项即可实现

## 系统自适应限流

[系统自适应限流 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/系统自适应限流)

系统规则支持以下的模式：

- **Load 自适应**（仅对 Linux/Unix-like 机器生效）：系统的 load1 作为启发指标，进行自适应系统保护。当系统 load1 超过设定的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护（BBR 阶段）。系统容量由系统的 `maxQps * minRt` 估算得出。设定参考值一般是 `CPU cores * 2.5`。
- **CPU usage**（1.5.0+ 版本）：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0），比较灵敏。
- **平均 RT**：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
- **并发线程数**：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
- **入口 QPS**：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。

## @SentinelResource

[注解支持 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/注解支持)

可以看到一般把这个注解添加到方法上。

如果我们使用`@SentinelResource`仅标注了资源名，那么当其被流控、熔断、系统保护...的时候就抛出异常`UndeclaredThrowableException`；而如果是本身的方法出现了问题那么会直接抛出对应的异常。

此时可以通过配置：

* `blockHandler`：当请求被`sentinel block`之后，调用的方法，需要注意的是，`blockHanlder`的值为处理的函数名称，这个函数必须是`public`类型的。且这个方法的参数列表在和原方法的参数完全一致的基础上，最后还需要再添加一个类型为`BlockException`类型的参数。且这个方法必须和当前的方法在一个类中（严重耦合）
* `fallback`：当手写的方法出现了异常（就是代码写的有问题），调用的方法，需要注意的是`fallback`的值为处理的函数名称，这个函数的方法返回值必须和原函数相同，这个方法的参数列表在和原方法的参数完全一致的基础上，最后可选的添加一个类型为`Throwable`类型的参数，表示异常。且这个方法必须和当前方法在一个类中（严重耦合）

现在考虑一个配置了流控，同时业务方法存在异常的情景，这里同时配置好了`blockerHanlder`和`fallback`，如果请求触发了流控，那么究竟应该调用那个方法？要注意的是对资源的`blockHandler`是大于`fallback`的，毕竟`blockHanlder`主管的是访问资源，如果被流控降级，那么根本就不会调用方法，从而触发`fallback`

去除耦合的方式是指定一个统一的异常类，分别配置：

* `blockHandlerClass`：当请求被`sentinel block`之后，会调用`blockHanlderClass`类的`blockHandler`的方法，需要注意的是方法类型必须为`static`
* `fallbackClass`：当手写的方法出现了异常（就是代码写的有问题），会调用`fallbackClass`类`fallback`方法，需要注意的是方法类型必须为`static`

一些其他的配置：

* `exceptionsToIgnore`：是一个`class`类型的数组，该字段下的异常类会被忽略。这里的忽略意味着他们不会进入`fallback`定义的函数中，但是出现了异常还是会被抛出

## 持久化

这里主要参考[动态规则扩展 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/动态规则扩展)

因为`sentinel`本身就是一个`jar`包，每次重启就发现上次配置的规则就没了，不仅如此，同一个项目，如果重新启动，`sentinel`中的配置也没了。所以需要对规则进行持久化。

下面主要是，将`sentinel`中的规则持久化到配置中心`nacos`中

首先需要引入依赖：

```xml
 <!--sentinel 规则持久化-->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

然后进行数据源的配置：

```yaml
spring: 
  cloud:
    sentinel:
      datasource:
        source:
          nacos:
            server-addr: 172.28.6.208:8848
            data-id: sentinel-config
            group-id: Sentinel_Flow_Control
            namespace: f6d71435-ecd1-479d-9ba6-ecc2df4cadc0
            data-type: json
            rule-type: flow
```

解释一下，首先我们配置数据源，肯定和`sentinel.datasource`相关

源码中相关的部分为：

```java
// 点进入看SentinelConstants.PROPERTY_PREFIX 就是sping.cloud.sentinel
@ConfigurationProperties(prefix = SentinelConstants.PROPERTY_PREFIX)
@Validated
public class SentinelProperties {
    public void setDatasource(Map<String, DataSourcePropertiesConfiguration> datasource) {
		this.datasource = datasource;
	}
}
```

可以看到，配置数据源，其实就是配置一个map，一个键值对，`key`这里可以随便起，不过`value`就有讲究了，进入`DataSourcePropertiesConfiguration`的源码看看：

```java
public class DataSourcePropertiesConfiguration {

	private FileDataSourceProperties file;

	private NacosDataSourceProperties nacos;

	private ZookeeperDataSourceProperties zk;

	private ApolloDataSourceProperties apollo;

	private RedisDataSourceProperties redis;

	private ConsulDataSourceProperties consul;
}
```

可以看到，它默认支持的配置中心还是很多的，不过这里因为我们使用的是`nacos`，所以也就仅配置一个`nacos`了

> 其实从这里也能看到，如果你想，甚至可以配置多个数据源，每次从多个配置中心获取配置。
>
> 不过一般的像nacos的都是push模式获取的配置，而redis这种就是pull模式获取
>
> 关于pull模式和push模式，这个可以看上面的github的wiki页面

然后进入`NacosDataSourceProperties`源码：

```java
public class NacosDataSourceProperties extends AbstractDataSourceProperties {

	private String serverAddr;

	private String contextPath;

	private String username;

	private String password;

	@NotEmpty
	private String groupId = "DEFAULT_GROUP";

	@NotEmpty
	private String dataId;

	private String endpoint;

	private String namespace;

	private String accessKey;

	private String secretKey;
}
```

这里面是它所有的属性了，可以看到，从简单的，`server-addr`，到`namespace`，`group-id`、`data-id`，都是有的，yaml文件的具体配置其实主要参考的还真是源码，上网上查，就只能告诉你应该配置什么，却不说为什么这么配，到`github`的`wiki`页面也没找到相关的配置，就只能看源码了。

这里面还少了上面的`data-type`和`rule-type`，其实这两个属性主要是继承来的，这里需要看源码`AbstractDataSourceProperties`

```java
public class AbstractDataSourceProperties {

	@NotEmpty
	private String dataType = "json";

	@NotNull
	private RuleType ruleType;

	private String converterClass;

	@JsonIgnore
	private final String factoryBeanName;

	@JsonIgnore
	private Environment env;
}
```

`data-type`这个就不说了，这个说的是配置文件的类型，默认就是`json`格式的，咱也不知道他支不支持其他格式的就先不改了

关键在于`rule-type`，这个是一个枚举类

```java
public enum RuleType {

	/**
	 * flow. 流控的配置
	 */
	FLOW("flow", FlowRule.class),
	/**
	 * degrade. 熔断降级的配置
	 */
	DEGRADE("degrade", DegradeRule.class),
	/**
	 * param flow. 热点参数的配置
	 */
	PARAM_FLOW("param-flow", ParamFlowRule.class),
	/**
	 * system. 系统自适应限流的配置
	 */
	SYSTEM("system", SystemRule.class),
	/**
	 * authority.
	 */
	AUTHORITY("authority", AuthorityRule.class),
	/**
	 * gateway flow.
	 */
	GW_FLOW("gw-flow",
			"com.alibaba.csp.sentinel.adapter.gateway.common.rule.GatewayFlowRule"),
	/**
	 * api.
	 */
	GW_API_GROUP("gw-api-group",
			"com.alibaba.csp.sentinel.adapter.gateway.common.api.ApiDefinition");
```

好吧，这里的配置类型已经比之前看到的都多了，问题不大，后面没见到过的，先放着不管。

然后就可以进入`nacos`的后台编写配置文件了，要注意的是文件类型要写成`json`格式。

现在的关键点在于`json`格式的配置文件应该怎么写，这个其实也没找到，网上就告诉说应该写`xxx`字段，也不知道为什么应该这么写，其实还是看源码，比如流控，我们`RuleType`枚举类看到了`FLOW`关联了一个配置类：`FlowRule`

```java
/**
 * Each flow rule is mainly composed of three factors: grade, strategy and controlBehavior.
 * The {@link #grade} represents the threshold type of flow control (by QPS or thread count).
 * The {@link #strategy} represents the strategy based on invocation relation.
 * The {@link #controlBehavior} represents the QPS shaping behavior (actions on incoming request when QPS exceeds the threshold)
 *
 * @author jialiang.linjl
 * @author Eric Zhao
 */

public class FlowRule extends AbstractRule {
	/**
     * The threshold type of flow control (0: thread count, 1: QPS).
     * 根据上面的图形化界面的内容，grade表示了阈值类型，是QPS还是线程数
     * 如果是0表示阈值类型为线程数，而1表示为QPS
     */
    private int grade = RuleConstant.FLOW_GRADE_QPS;

    /**
     * Flow control threshold count.
     * 特定类型阈值的大小
     */
    private double count;

    /**
     * Flow control strategy based on invocation chain.
     *
     * {@link RuleConstant#STRATEGY_DIRECT} for direct flow control (by origin);
     * {@link RuleConstant#STRATEGY_RELATE} for relevant flow control (with relevant resource);
     * {@link RuleConstant#STRATEGY_CHAIN} for chain flow control (by entrance resource).
     * 流控模式：直接、关联、链路，这里的RuleConstant就是一堆常量
     * 直连为0、关联为1、链路为2
     */
    private int strategy = RuleConstant.STRATEGY_DIRECT;

    /**
     * Reference resource in flow control with relevant resource or context.
     * 这个是关联的资源名，只有流控模式为关联或链路时，才考虑这个字段的意义
     */
    private String refResource;

    /**
     * Rate limiter control behavior.
     * 0. default(reject directly), 1. warm up, 2. rate limiter, 3. warm up + rate limiter
     * 流控效果：直接失败、warm up、排队等待
     * 此外我们还看到了warm up + 排队等待的组合，这个应该是如果在warm up的过程中请求达到了阈值，会排队等待
     */
    private int controlBehavior = RuleConstant.CONTROL_BEHAVIOR_DEFAULT;
	
    /**
     * 从这里也能看到，默认的预热时间为10s
     */
    private int warmUpPeriodSec = 10;

    /**
     * Max queueing time in rate limiter behavior.
     * 最大排队等待时间为500ms
     */
    private int maxQueueingTimeMs = 500;
    
    // 只要名字起的好，单位都不用愁，一看名字就知道预热的那个单位是秒，而排队的那个单位是毫秒
	
    /**
     * 是否是集群部署
     */
    private boolean clusterMode;
    /**
     * Flow rule config for cluster mode.
     */
    private ClusterFlowConfig clusterConfig;

    /**
     * The traffic shaping (throttling) controller.
     */
    private TrafficShapingController controller;

	// 注意到这里特意把toString方法拿出来，其实toString中对应的就是FlowRule中可以进行的全部配置了
    @Override
    public String toString() {
        return "FlowRule{" +
            "resource=" + getResource() +
            ", limitApp=" + getLimitApp() +
            ", grade=" + grade +
            ", count=" + count +
            ", strategy=" + strategy +
            ", refResource=" + refResource +
            ", controlBehavior=" + controlBehavior +
            ", warmUpPeriodSec=" + warmUpPeriodSec +
            ", maxQueueingTimeMs=" + maxQueueingTimeMs +
            ", clusterMode=" + clusterMode +
            ", clusterConfig=" + clusterConfig +
            ", controller=" + controller +
            '}';
    }
}
```

因为`FlowRule`继承了`AbstractRule`，在`toString()`方法中，还有两个属性`resource`和`limitApp`，说明这两个属性一定也是继承来的

```java
public abstract class AbstractRule implements Rule {

    /**
     * Resource name.
     * 这个就是资源名称
     */
    private String resource;

    /**
     * Application name that will be limited by origin.
     * The default limitApp is {@code default}, which means allowing all origin apps.
     * For authority rules, multiple origin name can be separated with comma (',').
     * 调用该资源的来源，如果写成default，表示不区分来源
     */
    private String limitApp;
}
```

举个例子，比如现在`controller`中的方法为：

```java
@SentinelResource(value = "hello")
@RequestMapping("/config/{name}/{age}")
public String hello(@PathVariable(value = "name", required = false) String name,
                    @PathVariable(value = "age", required = false) String age) {
    return "this is sentinel" + "name:" + name + " age:" + age;
}
```

那么我们的流控配置文件可以写成：

```json
[
    {
        "resource": "hello",
        "limitApp": "default",
        "grade": 1,
        "count": 1,
        "strategy": 0,
        "controlBehavoir": 0,
        "clusterMode": false
    }
]
```

针对资源`resource`进行流控，且阈值类型为`QPS`，配置每秒只能访问1次，达到阈值后访问会被拒绝

# Seata

官方文档在：[Seata 快速开始](http://seata.io/zh-cn/docs/user/quickstart.html)

## 一些基本概念

官网上给出了一个场景

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/seata_example.png)

这是一个网上支付的场景，用户购买商品逻辑由三个微服务构成：

- 仓储服务：对给定的商品扣除仓储数量。
- 订单服务：根据采购需求创建订单。
- 帐户服务：从用户帐户中扣除余额。

在这种情况下，本地的事务可以保证某一个服务的数据一致性，但因为服务的拆分，并不能保证购买商品，这一逻辑的数据一致性

seata就是为了解决分布式事务的一致性的

在github的项目界面[seata/seata: Seata is an easy-to-use, high-performance, open source distributed transaction solution. (github.com)](https://github.com/seata/seata)，中包含了对seata的描述

> seata的`github wiki`已经空了，全都需要去官网上看了

下面的内容来自`github`上的项目描述：分布式事务是一种全局事务，它是由一堆分支事务组成的，这些分支事务，可以认为是微服务的本地事务。

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/Global_Transaction.png)

在seata中具有三个重要的概念：

* **Transaction Coordinator(TC)：**维护全局和分支事务的状态，驱动全局事务提交或回滚。
* **Transaction Manager(TM)：** 定义全局事务的范围：开始全局事务、提交或回滚全局事务。
* **Resource Manager(RM)：**管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

全局事务的声明周期：

![](https://cdn.jsdelivr.net/gh/SunYuanI/img/img/seata_lifecycle.png)

> XID，即全局事务的ID

其主要流程如下：

* TM告知TC开启全局事务，TC生成全局事务ID（XID）
* XID被传播到微服务调用链
* RM向TC将本地事务注册为对应XID的全局事务的一个分支
* TM要求TC提交或回滚对应XID的全局事务
* TC要求对应XID的全局事务下的所有分支事务进行提交或回滚

## 部署

步骤在[Seata部署指南](http://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html)

因为前面提到了，`Seata`有三个重要的概念，下面的部署也是围绕着这三个部分：`TC`、`TM`、`RM`

这里面`TC`需要部署在`server`端，而`TM`和`RM`在`client`作为项目的依赖存在

### server端

存储模式默认为`file`，不过可选的修改为`db`或`redis`

这里如果需要使用`mysql`存储，需要在本地新建一个`database`，名字随便起。然后将它给出的`mysql.sql`中的表导入

> 如果下载的是`seata-server`的话其实是不带这个表的，需要下载源码，在其`script`子目录下，可以找到对应的`.sql`文件

然后进入`/conf`目录修改`file.conf`配置文件，选择将其`store.mode`修改为`db`，并修改下面的`db`的相关属性





其实这样就已经配完了，不过在seata中还可以进行高可用的配置，即从配置中心`nacos`中获取`seata`的配置

这里需要修改的是`/conf`目录下的`registry.conf`文件，这里因为需要选择的是`nacos`作为配置中心，所以进行如下的配置：

```properties
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = "f6d71435-ecd1-479d-9ba6-ecc2df4cadc0"
    cluster = "default"
    username = ""
    password = ""
  }
  # 后面的不重要
}
config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"

  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "f6d71435-ecd1-479d-9ba6-ecc2df4cadc0"
    group = "SEATA_GROUP"
    username = ""
    password = ""
    dataId = "seataServer.properties"
  }
  
  # 后面的不重要
}
```

所以最关键的就是将`registry`和`config`两个大标签下的`type`换成`nacos`，并将配置`nacos`的对应配置

最后进入`nacos`后台，添加配置，选择`dataId`默认应该为`seataServer.properties`

```properties
# 这里db和redis二选一
store.mode=db
# db的配置
store.db.datasource=druid
store.db.dbType=mysql
# mysql8以上的版本使用.cj包下的Driver
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=buzz
store.db.password=b
# redis的配置
store.redis.host=127.0.0.1
store.redis.port=6379
store.redis.maxConn=10
store.redis.minConn=1
store.redis.database=0
store.redis.password=null
store.redis.queryLimit=100
```



