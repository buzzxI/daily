# Controller

原来的 controller 还负责了页面跳转，不过现在前后端分离的情况下，页面的路由完全由前端负责，后端只需要用来获取数据就可以了

所以这里的 controller 都是标记了 @RestController 的，即都是以 JSON 格式返回数据

这里的 @RestController 可以理解为 @Controller 和 @ResponseBody 的组合，即一个 controller 内部的所有方法，都被 @ResponseBody 标记了

## [@ResponseBody](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-responsebody)

被 @ResponseBody 标记的方法，会将返回值通过 [HttpMessageConverter](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#rest-message-conversion) 进行序列化，并放入响应体(response body)中

这个 HttpMessageConverter 是一个转化器，用来实现 http request 和 response body 数据和 java 对象的转换，默认 spring boot 已经提供了很多的 converter，具体的可以看 [Message Conversion](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#rest-message-conversion)

>   注意到这里有一个 MappingJackson2HttpMessageConverter，这个转换器是用来将 json 数据和 java 对象之间的转化的，所以在写 rest controller 的时候，是不需要手动序列化对象的，交给 spring boot 去做就行

不过默认的 json 转化器可能不满足需求，在使用 spring boot 的时候，可以很方便的扩展原来的转换器列表，随便找个地方注册 bean 就行了，具体的可以看：[HttpMessageConverters](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.spring-mvc.message-converters)

在 spring mvc 中也提供了转换器的配置，具体的可以看 [Message Converters](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-message-converters)

而在 spring boot 中进行配置，只需要在 spring mvc 的配置类中实现方法 [configureMessageConverters()](https://docs.spring.io/spring-framework/docs/5.3.22/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html#configureMessageConverters-java.util.List-) 或 [extendMessageConverters()](https://docs.spring.io/spring-framework/docs/5.3.22/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html#extendMessageConverters-java.util.List-) 即可

>   关于 spring mvc 的配置类，spring boot 建议如果只需要修改其中一部分配置的话，就自己新建一个配置类，实现 WebMvcConfigurer 接口，添加 @Configuration 注解，实现对应的方法即可，注意，不要添加 @EnableWebMvc 注解，因为这个注解会取消 spring boot 的默认配置，这样的话，所有的配置都需要手动写了

这两个方法有的时候还是比较容易混淆的，区别在于一个是配置原始的转换器，一个是用来添加新的转换器

configureMessageConverters()，这个方法是用来修改原始的默认转换器的，如果在这个方法中添加了 MappingJackson2HttpMessageConverter，那么默认的转换器就不见了

extendMessageConverters()，这个方法是用来扩展原始的转换器列表的，即如果在这个方法中添加了 MappingJackson2HttpMessageConverter，原始的转化器依旧存在

注意因为受限于 spring 处理 http 请求的方式，在运行期间会保存一个转换器列表(list)，spring 会通过遍历整个列表，找到可以处理当前 request body 的转换器，如果找到了，就不会继续向后查找，因此一定要注意转换器的顺序关系

## [@RequestBody](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestbody)

@RequestBody 用来将 request 的 body 序列化成一个 java 对象，当然也是通过上面提到的 HttpMessageConverter

## [@CookieValue](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-cookievalue)

这个是用来获取请求中的 cookie 的，需要配置一个 value，表示需要获取的 cookie 的名称，比如：

```java
@GetMapping("/test")
public void test(@CookieValue("test_token") String testToken) {
    log.info(testToken);
}
```

当然使用这个注解获取 cookie 一定要一并设置好其异常处理，不然返回的 HttpStatus 就不是 200 了

>   你别管出现什么问题，总得让用户收到一个正常的响应吧

如果对应名字 cookie 没有在 request 中出现，那么这里会抛出：org.springframework.web.bind.MissingRequestCookieException

这里要主动处理的异常就是这个，这个类提供了一个方法 getCookieName()，可以获取 cookie 名字，用来自定义的异常处理

## @ResponseStatus

一般的，如果希望修改 HTTP 响应的状态码，可以通过在 controller 的特定 handler 中添加一个 response 参数，修改状态码

如果不希望给方法添加参数，可以借助这个注解返回不同的状态码，通过在方法上添加这个注解，可以直接修改返回的状态码，比如：

```java
@PostMapping("/login")
@ResponseStatus(HttpStatus.ACCEPTED)
public UserVo login(@RequestBody UserVo userVo) {
    log.info(userVo.getUsername());
    log.info(userVo.getPassword());
    return userVo;
}
```

这里的 HttpStatus.ACCEPTED 对应的状态码为 202，而不是经典的 200(OK)

有的时候可能需要返回 400 系列的状态码，表示请求错误，显然，只有出现错误的时候才会这么返回，并不能直接在方法上直接添加注解，此时可以通过抛出异常的方式返回状态码

比如现在自定义了异常类 IllegalAccessException

```java
package icu.buzzx.blogspring.exceptions;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

/**
 * @author buzz
 * @date 2022/9/15 16:03
 * @description TODO
 */
@ResponseStatus(code = HttpStatus.UNAUTHORIZED, reason = "未拥有访问权限")
public class IllegalAccessException extends RuntimeException {
}
```

注意到使用了注解 @ResponseStatus 进行标注，此外还表明了原因，这种标注了 @ResponseStatus 的运行时异常，会被 ResponseStatusExceptionResolver 处理，所以只管抛出，不用 catch

## MultipartFile

如果有文件上传的需求的话，在后端使用 MultipartFile 接收文件

一般的前端会通过 FormData 的形式将文件传递到后端，为了获取 key 对应的文件，可以通过 @RequestParam 指定 key 的名称

>   其他需求的话，还可以使用 @RequestPart 指定 key

## 抽取 controller

有的时候访问的路径具有相同的前缀，比如前台页面的请求可能都以 /front 作为前缀，后台页面的请求可能都已 /management 为前缀

此时可以通过抽取 controller 的形式，统一分配 @RequestMapping，但要注意的是，一定要定义接口，而不是某个实现类

```java
// ManagementController.java
package icu.buzzx.blogspring.controller;

import org.springframework.web.bind.annotation.RequestMapping;
/**
 * @author buzz
 * @date 2022/11/2 10:23
 * @description 所有后台的请求的前缀都包括了 /management，因此抽象出来
 * 注意注解 @RequestMapping 是可以抽取出来的，@RestMapping 不太行
 */
@RequestMapping("/management")
public interface ManagementController {}
```

这样所有实现了这个接口的 controller 默认就已经携带了路径前缀 /management

但要注意的是，如果实现类添加了注解 @RequestMapping，那么从接口中继承的路径会被覆盖

# [Configuration Property](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties)

有的时候程序需要从配置文件中获取对应的配置

## @Value

使用 `@Value` 是最简单的获取配置的方式

```java
@Value("${my.service.name}")
String serviceName;
```

只要 serviceName 是某个 bean 的成员变量, 那么在运行阶段 serviceName 会被注入 `application.yaml` 中的 my.service.name

## [@ConfigurationProperty](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties.java-bean-binding)

一种更好的管理方式是使用某个类保存所有需要注入的属性, 并按需添加该类的成员变量

官方给出了一个很好的例子: 

```java
import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("my.service")
public class MyProperties {
	// 对应 my.service.enable 
    private boolean enabled;
	// 对应 my.service.remote-address
    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() {
        return this.enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public InetAddress getRemoteAddress() {
        return this.remoteAddress;
    }

    public void setRemoteAddress(InetAddress remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    public Security getSecurity() {
        return this.security;
    }

    public static class Security {
		// 对应 my.service.security.username
        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() {
            return this.username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String getPassword() {
            return this.password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        public List<String> getRoles() {
            return this.roles;
        }

        public void setRoles(List<String> roles) {
            this.roles = roles;
        }

    }
}
```

默认的, spring 会通过 getter/setter 注入 bean

### enable property

@ConfigurationProperty 默认不会将当前的类注入为 spring 的 bean, 所以只添加这个注解, 这个配置类还是用不了

一种比较简单的使能方式是为当前配置类添加 `@Component` 注解, 直接注册为 bean

一种比较正规的做法是使用注解 `@EnableConfigurationProperties`, 并填入对应需要使能的配置类, 比如: 

```java
@SpringBootApplication
@EnableConfigurationProperties({ApplicationProperty.class})
public class WebApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class, args);
    }
    
}
```

上面在最顶层的文件中使能了配置类 `ApplicationProperty`

不过在一般的开发中, 可能会写很多的配置类, 对 springboot 本身的功能进行自定义, 比如实现接口 `WebMvcConfigurer`, 所以一般可以在某个配置类上添加 `@EnableConfigurationProperties` 以使能配置类

## 适配 IDEA

写好了配置类, 在 yaml 中填入配置的时候默认是没有高亮的, 此时就需要添加一个注解处理器

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

具体的文档可以看 [官网](https://docs.spring.io/spring-boot/docs/2.7.3/reference/html/configuration-metadata.html#appendix.configuration-metadata.annotation-processor), 官方上来就说了, 这个注解处理器就是为 IDE 的自动补全服务的

如果项目中还是用了 lombok (一个插件，也是依赖于注解处理器的)，那么此时，要保证 lombok 的注解处理器运行在 spring-boot-configuration-processor 之前，为了实现这点，官方要求：

*   如果使用 maven-compile-plugin，那么在 [<annotationProcessors>](https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#annotationProcessors) 标签中应该 lombok 的注解处理器排在前面
*   如果不使用上面的插件，那么可以在添加依赖的时候，把 lombok 的依赖放在 spring-boot-configuration-processor，之前，因为默认的话就是按照 dependency 的顺序安排注解处理器的

==**官网还说了和 Aspectj 同时使用的时候需要保证注解处理器仅运行一次，这里先挖一个坑，后面遇到了再说吧**==

# 静态资源共享

自从将前后端分离了，后端就再也不需要管 .js，.css 的静态资源文件了，这里的静态资源文件，最典型的就是图片文件

## 经典的 spring mvc 配置

在 spring mvc 中可以通过配置 ResourceHandler 实现静态资源访问 -> [spring mvc static resource](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-static-resources)

默认的 springboot 装配的静态资源访问路径即为根目录，即如果存在资源 test.png 那么访问路径为：http://localhost:8080/test.png

且静态资源需要放在 /static 下(还有一些其他的默认路径，这些都不重要，因为默认的目录都是原来放静态 css，js 文件的时候使用的，现在前后端分离了，还是不要将文件和项目放在一个目录下了)

所有的关于 spring mvc 的配置，都是通过实现 WebMvcConfigurer 接口实现的

>   还是要说一遍，因为使用 springboot 进行了自动装配，注意不要在 mvc 的配置类上添加注解 @EnableWebMvc 这将意味着所有的自动配置都失效了，都需要重新自己配置

关于静态资源，需要实现接口下的 addResourceHandlers 方法

```java
/**
 * 添加静态资源访问路径的配置
 */
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/img/**")
        .addResourceLocations("file:///F:/uploaded/");
}
```

比如上面配置了访问路径为：根目录/img，且实际将文件存放在了本地的文件系统中

>   在 win 中文件系统需要写成：file:///；而在 linux 中写成：file:/
>
>   其他的写法不一定有问题，但上面这种写法最保险

关于路径的写法：访问路径需要写成：/img/** 后面的 /** 比较关键，而文件存储路径中最后的一个 '/' 很关键，如果缺少这个路径，就会不解析

## springboot 自动装配

使用 springboot，关于 mvc 的配置就可以从 yaml 配置文件中配置了，当然也就包括了静态资源的访问路径

比如还是上面的配置，在 application.yaml 中可以写成：

```yaml
spring:
  mvc:
    static-path-pattern: /file/**
  web:
    resources:
      static-locations: file:///F:/uploaded
```

注意到在写文件存放路径的时候，并不是以 '/' 结尾的，但 spring mvc 还是可以正常解析资源，如果看源码的话可以看到，springboot 在获取到配置后会调用 appendSlashIfNecessary 方法，即会在结尾缺少 '/' 的时候自动补上一个 '/'

## ResourceResolver

其实所有的访问静态资源的请求都是通过 ResourceResolver 解析的，看上面 spring mvc 重写了 addResourceHandlers 方法，但并没有指定resource resolver 的类型，其实默认的就是 PathResourceResolver()，即路径匹配的的 resource resolver

# MessageConverter

spring boot 自动装配了很多 message converter，并将所有的转化器放入了一个 list 中，每一个转换器都和一个 MIME TYPE 关联

当一个新的请求到达时，spring boot 会遍历整个 list，判断某个转换器是否可以当前这种类型的报文，当然，判断依据为转换器关联的 MIME TYPE 

特别的 springboot 会选择 MINE TYPE 和请求报文的 content-type 字段匹配的转换器用来处理 request，在返回响应的时候，会使用 MINE TYPE 和请求报文的 accpet 字段匹配的转换器填充 response 的 body

# 针对 spring mvc 的异常处理

主要就是针对在 controller 部分抛出的异常处理

## 异常处理器

类似于 Message Converter，spring boot 默认自动专配了很多异常处理器

所有的异常处理器都实现了接口 HandlerExceptionResolver，具体的装配的处理器可以看 [HandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-exceptionhandlers)

### DefaultHandlerExceptionResolver

这个异常处理器用来映射异常到某个 Http 状态码，具体的映射规则可以看 [DefaultHandlerExceptionResolver (Spring Framework 5.3.23 API)](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html)

这个映射其实很挺准确的，只不过问题在于不能自定义，所以才叫 Default...Resolver

### ResponseStatusExceptionResolver

这个前面提高 [@ResponseStatus](#@ResponseStatus) 的时候提到过，首先自定义异常，然后抛出这个异常，就可以返回对应的 Http 状态码的报文，此外如果在 @ResponseStatus 中添加了 reason 字段，那么在 response body 中还包含了错误信息，很人性化

不过这个处理器也并不是很智能，还是一样的，无非就是添加了状态码和message，对于 response 报文，还是没有完全的控制权

### 自定义 ExceptionResolver

没这个必要，太麻烦了，还需要返回 ViewModel，虽然对于 response 报文具有完全的支配权，不过还是有点麻烦了

## @ResponseStatus & ResponseStatusException

这两个东西其实差不多，第一个 @ResponseStatus [之前](#@ResponseStatus)就已经说过了，这里重点关心后面的这个异常类，这个异常类用起来也很简单

使用的时候直接在 controller 中抛出这个异常即可，构造一个 ResponseStatusException，需要一个 Http 状态码，可选参数包括一个出现异常的原因和一个出现的具体异常

```java
@GetMapping(value = "/{id}")
public Foo findById(@PathVariable("id") Long id, HttpServletResponse response) {
    try {
        Foo resourceById = RestPreconditions.checkFound(service.findOne(id));

        eventPublisher.publishEvent(new SingleResourceRetrievedEvent(this, response));
        return resourceById;
     }
    catch (MyResourceNotFoundException exc) {
         throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Foo Not Found", exc);
    }
}
```

通过简单的 throw 就可以返回一个带有指定状态码和指定内容的 http response

所以使用的使用其实和 @ResponseStatus 十分类似，毕竟从异常响应报文的形式上就基本上一致

这种类型的异常处理已经完成了一部分报文自定义，但问题在于在 response 的 body 中还是包含了很多其他的信息，无论是通过注解还是通过 throw ResponseStatusException 都只能定义一部分的 response body，而不能完全控制

## @RestControllerAdvice

配合上注解 @ExceptionHandler 可以实现全局的异常处理，这也是处理 spring mvc 异常的最终形式

对于异常控制，如果希望完全掌握整个过程，可以通过自定义异常处理器实现，不过这样做的问题在于代码表现形式上不是很 rest，需要返回一个 ModelAndView

后面出现的注解 @ResponseStatus 和 ResponseStatusException 通过几行代码就可以修改 response 中的状态码，但无法完全控制 respose body

这里说的 @RestControllerAdvice + @ExceptionHandler 可以实现全局且完整的异常控制

先看看 [@ExceptionHandler](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice) 单独来看的话这是一个用在 controller 内部，处理异常的

再看一下官方对 @ControllerAdvice 的解释 [@ControllerAdvice](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)，可以认为是一个针对所有 controller 的切面，在一个类配置了 @RestControllerAdvice 的类中添加了 @ExceptionHandler，即可完成对所有 controller 的异常配置

因为 @ExceptionHandler 标注的方法中可以返回一个 ResponseEntity，这意味这，可以实现对 response 的 header、status、body 的完全掌控

```java
@RestControllerAdvice(annotations = RestController.class)
public class WebExceptionHandler {

	@ExceptionHandler(IllegalAccessException.class)
	public ResponseEntity<String> handleIllegalAccess(IllegalAccessException e) throws JsonProcessingException {
		ObjectMapper mapper = new ObjectMapper();
		return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(mapper.writeValueAsString(ResponseData.error(100, e.getMessage())));
	}
}
```

在 body 中写入了一个 ResponseData，这是一个实体类，用来表示返回数据：

```java
@Data
public class ResponseData<T> {
	private int code;
	private String msg;
	private Collection<T> data;

	private ResponseData() {}

	public static <T> ResponseData<T> success() {
		ResponseData<T> responseData = new ResponseData<>();
		responseData.code = 200;
		responseData.msg = "OK";
		return responseData;
	}

	public static <T> ResponseData<T> success(Collection<T> data) {
		ResponseData<T> responseData = success();
		responseData.data = data;
		return responseData;
	}

	public static <T> ResponseData<T> error(int code, String msg) {
		ResponseData<T>  responseData = new ResponseData<>();
		responseData.code = code;
		responseData.msg = msg;
		return responseData;
	}
}
```

而这个捕获的异常 IllegalAccessException 也是一个自定义的异常

```java
public class IllegalAccessException extends RuntimeException {

	public IllegalAccessException() {
		this("未授权的访问");
	}
	public IllegalAccessException(String msg) {
		super(msg);
	}
}
```

这样配合下来，在 controller 中如果遇到了异常，那么直接 throws 一个 exception，随后异常会被 WebExceptionHandler 捕获，并在方法 handleIllegalAccess 得到处理，最后返回一个 response 实体，这个实体的 status 和 body 都是统一配置的

# [拦截器](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-handlermapping-interceptor)

简单来说，为了实现拦截器的功能，需要实现接口 HandlerMapping，具体的就是实现这个接口的三个方法

*   preHandle：Before the actual handler is run(就是请求进入 controller 之前，这个方法具有一个返回值为 boolean 类型，表示是否放行请求)
*   postHandle: After the handler is run(不用管，这个是在返回 ModelAndView 之前调用，因此操作的应该是数据)
*   afterCompletion: After the complete request has finished(这个时候请求已经从 controller 离开了)

这里面最关键的其实就是 preHandle 方法，比如用于参数校验，如果校验不通过就不让请求进入 controller

注意到这个 preHandle 方法中提供了参数 response，因此可以直接对请求进行操作

因为 spring 的核心思想就是 bean，所以显然一个拦截器需要被注册为 bean，这里的注册的方式就是在 WebMvcConfigure 中重写 addInterceptors 方法

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
	/**
	 * 这里的两个拦截器就是两个例子，不同太关系具体是什么，重点在于通过 new 的方式向 registry 注册了 bean
	 */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
    }
}
```

# Data

## Druid

首先不管使用什么框架都需要一个数据库连接池，默认的 springboot [默认](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.datasource.connection-pool)使用 Hikari 作为数据源

这里使用 druid 和性能什么都不沾边，纯粹是为了配置而配置，默认的数据库连接池已经足够强大了，也许国内的企业都是喜欢这种国产开源产品呢

>   个人认为 druid 强大的地方在于监控和安全性

### 基础配置

无论什么数据源，老四样都需要配置：

*   driveClassName
*   url
*   username
*   password

而因为使用了 druid 作为了默认的数据源，所以还需要额外配置 datasource 的 type 属性

```yaml
spring:
  # 配置数据库连接的基本信息，即老四样
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/blogs?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf8&useSSL=false
    username: root
    password: root
    # 使用 Druid 数据库连接池作为数据源
    type: com.alibaba.druid.pool.DruidDataSource
```

### 特有的配置

主要是监控安全性的配置

## Mybatis

使用 mybatis 一切以[官网](https://mybatis.org/mybatis-3/zh/getting-started.html#)为准

因为有的时候使用的是 mybatis 和 spring 结合的性质，比如配置 bean，这个时候看 [mybatis-spring](https://mybatis.org/spring/zh/getting-started.html)

当然，因为主要使用的还是在 springboot 中使用 mybatis，所以 [mybatis-spring-boot-starter](https://github.com/mybatis/spring-boot-starter/blob/master/mybatis-spring-boot-autoconfigure/src/site/zh/markdown/index.md) 重要性其实比 mybatis-spring 更高

### 基础配置

为了在 springboot 中使用 Mybatis，首先需要从 [maven 仓库](https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter) 导入 mybatis-spring-boot-starter 依赖

>   mybatis 的启动器
>
>   因为是自己用，所以建议使用最新版，每次版本都不太一样，所以最好还是每次上 maven 仓库查一下

```xml
<!-- mybatis-springboot 的依赖 -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>
```

在 mapper 中写接口就不说了，总之使用了 Mybatis 就不需要写 dao(mapper) 的实现类了

然后编写 mybatis 的配置文件

```yaml
# application-dev.yaml
mybatis:
  # 别名所在的包，好处是 parameterType 和 resultType 不需要写全类名了
  type-aliases-package: icu.buzzx.pojo
  # 存储 mapper.xml 的路径
  mapper-locations: classpath:mappers/*.xml
  # 开启数组库中下划线到实体类中驼峰命名的转换
  configuration:
    map-underscore-to-camel-case: true
```

>   基础的配置不算多，但注意到所有的配置都写在了 springboot 的 yaml 配置文件中了，以后也是这样，因为使用了 springboot 就不要让配置文件满天飞了，反正一个 yaml 中都可以放下，为什么不放在这里呢

### 映射配置文件

首先一个 mapper.xml 具有如下的基本框架：

```xml
<!-- UserMapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--Mybatis的映射配置文件-->
<mapper namespace="${mapper信息}">
</mapper>
```

根据官网对 namespace(命名空间)的解释，作用有两个：

*   隔离不同的 sql 语句
*   实现接口绑定

>   个人认为，这里 namespce 的主要是用来实现接口绑定的，接口内部定义了不同的方法，用来实现 CRUD 操作
>
>   所以不同的接口之间 sql 语句本身就是相互隔离的

所以一般情况下，一个 mapper 接口会和一个 mapper.xml 文件是一一对应的，在 mapper.xml 中通过 namespace 绑定了特定的 mapper 接口

而在一个映射配置文件中，需要实现对应 mapper 接口中方法的 sql 语句，完成 sql 的 CRUD 操作

### 语法问题

在 mapper.xml 中不仅仅需要写 sql 语句，更需要写实体类和数据库中属性的对应关系，对于复杂查询将数据库中的表结构和 java 对象的映射可能不是十分容易，所以要把映射配置文件的的语法单独拎出来说一下

#### [select](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#select)

查询语句，一般的话会根据一些参数，返回某个对象

比如根据 username 和 password 查询 user

```xml
<!-- UserMapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--Mybatis的映射配置文件-->
<mapper namespace="icu.buzzx.mapper.UserMapper">
    <select id="getUserByUsernameAndPassword" parameterType="UserVo" resultType="User">
        select * from t_user where username = #{username} and password = #{password}
    </select>
</mapper>
```

注意到上面给出了一个映射配置文件(UserMapper.xml)

除开上面提到的 .xml 文件约束之外，最内层的 select 标签包含了真正用于查询的 sql 语句

一个 select 标签的 id 是这个标签在命名空间中的唯一标识符，用来标识一条 sql 语句，一般的 id 和特定 mapper 接口下的需要实现的方法名保持一致

>   其实可以认为 xxxMapper.xml 实现了特定 mapper 接口，至少是实现了接口中方法的 sql 语句

一个 select 查询从数据库中返回一个表，而实际 java 中使用对象进行数据封装，显然这里就涉及到了对象和表结构之间的转换，这部分也需要重点配置

select 标签中使用 parameterType 表示传入参数的对象类型，resultType 表示了 sql 查询返回的对象类型

>   如果在 mybatis 的配置中添加了 type-aliases-package 那么这里就可以写类名；否则这里应该写全类名，不然可能会报错

在 sql 语句中使用了两个符号 #{username} 和 #{password}，这是 mybatis 的占位符，用来传递参数，具体的可以看[占位符](#占位符)

### [占位符](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#字符串替换)

在 mybatis 中具有两种占位符：${} 和 #{}，其中前者仅仅是 sql 字符串的拼接，而使用后者 mybatis 会创建 PreparedStatement 参数占位符，可以有效的避免 sql 注入的问题

所以可以认为基本上在所有的应用场景中，都应该考虑优先使用 #{} 作为占位符

不过 ${} 并不是毫无用处，官网给出了一个很好了例子，这里仅仅是照搬官网查询用户的例子：

```java
@Select("select * from user where id = #{id}")
User findById(@Param("id") long id);

@Select("select * from user where name = #{name}")
User findByName(@Param("name") String name);

@Select("select * from user where email = #{email}")
User findByEmail(@Param("email") String email);

// 其它的 "findByXxx" 方法
```

可以看到这三个方法具有很多的重复部分，他们都是使用了某个条件进行用户查询，可以将这些语句简化为：

```java
@Select("select * from user where ${column} = #{value}")
User findByColumn(@Param("column") String column, @Param("value") String value);
```

这里列名是动态变化的，作为参数传入，而实际在执行的时候 ${column} 会被直接替换掉，而 #{value} 会被替换为 ？

注意这里的 ${column} 是存在 sql 注入的问题的，所以一定不要让这个参数直接绑定用户输入。具体的可以为这个字段添加校验

### TypeHandler

#### 一般的 typehandler

自定义类型转换器，这里结合 springboot 使用存在一定的问题，暂时还不能"优雅的"解决，不过找到了一种相对麻烦一点的配置方式，可以解决这个问题

有关的资料全部参考了官网：[mybatis – typehandler](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)

简单来说，要实现一个自定义的 typehandler，需要：

*   继承 BaseTypeHandler，这是一个泛型类，需要泛型类型即为 javaBean 的类型
*   添加注解 @MappedTypes，并在注解中指明映射到的 java 类型：即 value 为 xxx.class
*   添加注解 @MappedJdbcTypes，并在注解中指明映射到的数据库中的类型：即 value 为 JdbcType.xxx (一个常量)

BaseTypeHandler 是一个抽象类，需要实现这个抽象类的四个方法，这里以 java type 为 boolean 映射到 jdbc type 为 integer 为例：

```java
package icu.buzzx.blogspring.handlers;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;
import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * @author buzz
 * @date 2022/10/12 11:39
 * @description 将 java 中的 boolean 类型转换为 mysql 中的 int 类型
 */
@MappedJdbcTypes(JdbcType.INTEGER)
@MappedTypes(Boolean.class)
public class BoolValueTypeHandler extends BaseTypeHandler<Boolean> {

	@Override
	public void setNonNullParameter(PreparedStatement ps, int i, Boolean parameter, JdbcType jdbcType) throws SQLException {
		int val = parameter ? 1 : 0;
		ps.setInt(i, val);
	}

	@Override
	public Boolean getNullableResult(ResultSet rs, String columnName) throws SQLException {
		return rs.getInt(columnName) != 0;
	}

	@Override
	public Boolean getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
		return rs.getInt(columnIndex) != 0;
	}

	@Override
	public Boolean getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
		return cs.getInt(columnIndex) != 0;
	}
}
```

三个 get 方法表示通过 ResultSet 从数据库中获取数据后，将 jdbc type 转化为指定 java type 的方法，最上面的 set 方法，表示通过 PreparedStatement 向数据库中添加数据时将 java type 转化为 jdbc type 的方法

>   如果之前使用过原生的 jdbc API 进行 sql 语句查询的话，应该知道 ResultSet 和 PreparedStatement

然后就是天坑的地方了，大多数资料都会说，在 mybatis-config.xml 文件中添加 typehandler，但因为这里是 springboot 环境，所以这里在 yaml 文件中添加全局配置：

```yaml
mybatis:
# ... 若干其他配置
  type-handlers-package: [这里添加存放了映射器的包名]
```

然后很多教程说，就可以直接使用了，因为前面指定了 java type 和 jdbc type，后面如果 sql 操作的时候遇到了对应的类型，就会使用自定义的 typehandler 进行映射

然而实际上根本就没有这么好用，如果只进行了全局配置，那么在 sql 操作的时候还是不会使用自定义的 typehandler

目前看来，唯一有效的方法，就是在 xxxMapper.xml 中，配置上指定的 typehandler

*   如果是 select 语句，就在 ResultMap 中的 \<id> 或者 \<result> 中配置上指定的 typehandler(指写上全类名)，这意味着 select 语句必须使用 ResultMap 做结果映射
*   如果是 update、insert、delete 语句，就在占位符中 #{} 添加上 typehandler

这里相当于给某些字段指定了 TypeHandler，写起来十分的繁琐，不是很优雅，不过目前也只有这个方法可以是实现指定映射配置

#### 枚举类的 typehandler

在具体解释如何使用 Enum 的 typehandler 之前，最好先看一下[如何使用枚举](#使用枚举)

我自己的项目中，一般枚举类都具有两个类型的属性 string 类型表示枚举对象的取值，int 类型表示一个编号，主要是将枚举对象存入数据库的时候可能需要使用到

这里为了构造一种具有通用的枚举类处理器，参考官方文档，为 typehandler 配置了一个构造器，表示在new 一个 typehandler 的时候，将 typehandler 处理的对象类型作为参数传入

```java
//GenericTypeHandler.java
public class GenericTypeHandler<E extends MyObject> extends BaseTypeHandler<E> {

    private Class<E> type;

    public GenericTypeHandler(Class<E> type) {
        if (type == null) throw new IllegalArgumentException("Type argument cannot be null");
        this.type = type;
    }
    ...
}
```

特别的对于通用的枚举类处理器，形式应该如下：

```java
package icu.buzzx.blogspring.handlers;

import icu.buzzx.blogspring.constant.IntValueEnum;
import icu.buzzx.blogspring.constant.PostState;
import icu.buzzx.blogspring.constant.StringValueEnum;
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;

import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * @author buzz
 * @date 2022/10/12 21:32
 * @description 枚举类的类型转换器
 * 因为我定义的枚举类都会实现两个接口
 */
@MappedTypes(PostState.class)
@MappedJdbcTypes(JdbcType.INTEGER)
public class EnumTypeHandler<E extends IntValueEnum & StringValueEnum> extends BaseTypeHandler<E> {
	// 某种 typehandler 就对应了某种枚举类，这种枚举类会在映射的时候确定
	private E[] constants;
    // 构造器传入枚举类型并初始化成员变量 constants
	public EnumTypeHandler(Class<E> type) {
		this.constants = type.getEnumConstants();
	}
	// 根据数据库中查询的结果获取枚举类
	private E getConstantByInt(int val) {
		for (E constant : constants) {
			if (constant.getIntValue() == val) return constant;
		}
        // NoSuchEnumException 是自定义的异常类，继承于 ServerInternalException，表示服务器本身的异常
        // 在 controller 层会有专门异常处理器捕获 Server...Exception
		throw new NoSuchEnumException(constants.getClass() + "找不到枚举类:" + val);
	}
	// 所有继承了 BaseTypeHandler 的 TypeHandler 都需要实现的四个方法
	@Override
	public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
		ps.setInt(i, parameter.getIntValue());
	}

	@Override
	public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
		int val = rs.getInt(columnName);
		return getConstantByInt(val);
	}

	@Override
	public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
		int val = rs.getInt(columnIndex);
		return getConstantByInt(val);
	}

	@Override
	public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
		int val = cs.getInt(columnIndex);
		return getConstantByInt(val);
	}
}
```

其实枚举类的类型处理器，和一般的类型处理器差不多，区别在于，我为了实现通用枚举类处理器，需要定义一个构造方法，并将枚举类传入，将所有的枚举类对象作为成员变量保存，并定义一个 getConstant 方法，试图在从数据库中查询到 int 类型时，将其转化为某个枚举类型

### PageHelper

mybatis 的分页插件，项目地址在 [pagehelper/Mybatis-PageHelper: Mybatis通用分页插件 (github.com)](https://github.com/pagehelper/Mybatis-PageHelper)

>   简化了分页查询的复杂操作

因为官方已经提供了 spring-boot 的 starter，因此在 pom.xml 中添加依赖后，直接可以在 yaml 文件中进行配置

在配置文件中，可以对 pagehelper 进行很多默认配置，其中常用的一些：

```yaml
pagehelper:
  # 默认 pagehelper 会检查当前的数据库链接，选择合适的分页语法
  # 为了避免 pagehelper 自身检测的问题，手动配置语法为 mysql
  helper-dialect: mysql
  # 参数合理化，即如果查询的页数 page <= 0 那么会查询第一页
  # 如果 page > 最大页数，就会查询最后一页
  reasonable: true
```

pagehelper 会在分页查询的时候进行拦截，并添加查询页数和每页大小的条件，所以使用的时候完全不需要修改 sql 语句，比如如果需要分页查询文章列表，之需要写成：

```java
public List<Article> getArticleListByPage(int page, int pageSize, String uuid) {
    PageHelper.startPage(page, pageSize);
    return articleMapper.getArticleListByUuid(uuid);
}
```

这是 pagehelper 最常用的用法，调用方法 startPage，并将需要查询的页数和分页大小作为参数传入，而在 mapper 进行查询的时候，完全不需要修改原来的 sql 语句，还是进行全局查询，pagehelper 会帮我们完成分页 sql 语句的拼接(拦截后拼接)

# 使用枚举

习惯上把枚举类看成一个常量，这里我个人习惯上让一个枚举类保存两个类型的 val：string 类型和 int 类型

其中 int 类型会被存储到数据库；string 类型会在 json 序列化时为对应字段赋值，响应给前端

我的枚举类都放在了 /constants 包下，首先定义了两个接口：
```java
package icu.buzzx.blogspring.constant;

/**
 * @author buzz
 * @date 2022/10/12 21:23
 * @description 所有包含了 string 类型的枚举类都需要实现这个接口
 */
public interface StringValueEnum {
	String getStringValue();
}
```

```java
package icu.buzzx.blogspring.constant;

/**
 * @author buzz
 * @date 2022/10/12 21:33
 * @description int 类型的枚举类都需要实现这个接口
 */
public interface IntValueEnum {
	int getIntValue();
}
```

这两个接口表示了两种行为，即获取枚举类的 int 类型和获取枚举类的 string 类型

>   其实我认为将枚举类当成实体类也是可以的，使用 lombok 为枚举类的所有属性自动配置上 getter 方法
>
>   不过个人习惯了使用接口限制行为

比如我要定义一个枚举类：PostState，它具有两个状态，已经发布，和草稿状态，那么可以写成：

```java
package icu.buzzx.blogspring.constant;

/**
 * @author buzz
 * @date 2022/10/12 21:24
 * @description 文章的发布状态
 */
public enum PostState implements StringValueEnum, IntValueEnum{

	/**
	 * 草稿状态
	 */
	DRAFT("draft", 0),
	/**
	 * 已发布状态
	 */
	PUBLISHED("published", 1);

	private String value;
	private int intValue;

	PostState(String value, int intValue) {
		this.value = value;
		this.intValue = intValue;
	}

	@Override
	public String getStringValue() {
		return this.value;
	}

	@Override
	public int getIntValue() {
		return this.intValue;
	}
}
```

# spring security & jwt

整体分为两部分: authentication (认证) 和 authorization (授权)

一般而言 jwt 包含了用户的身份信息, 可以被认为是一个用户凭证, 整体流程如下:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/03/02/21:06:58:jwt_authentication.png)

当用户登录后, server 会向 client 返回一个表明了用户身份的 jwt (存放在 header 中); 后续 client 的请求, 只要携带了该 jwt 就表明了用户的身份

一般而言 server 可以根据 jwt 对用户行为进行控制, 限制用户的访问: allow access to the resource or denie

jwt 即 json web token, 本身由三部分组成: header, payload, signature

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/03/02/21:39:51:jwt_structure.png)

其中 header 和 payload 是明文信息, 在 jwt 中仅仅进行了编码, 没有加密; 而 signature 部分是有前两部分经由 secret key 加密得到的, 该 secret key 应该保存在 server 端 (不能被 client 得知) 

signature 部分并不能对 header 和 payload 进行加密, 其更多的作用是数字水印, 当 server 收到了来自 client 的一个 jwt 后, 会再次对 header 和 payload 通过 secret key 进行加密, 并将其和 signature 进行校验, 只有校验通过的 jwt 才会被 server 认可

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/03/02/21:47:07:jwt_validation.png)

一般而言 secret key 是随机生成的, copilot 给出了一个 python 程序用来生成该 key

```python
import secrets

# Generate a random JWT secret key (32 bytes)
jwt_secret_key = secrets.token_hex(32)

# Print the generated secret key
print(f"Generated JWT secret key: {jwt_secret_key}")
```

上述流程其实不重要, 因为在最新的 jjwt 验证中, 已经弃用了这种方式, 而强制要求用户必须使用 private key 和 public key pair; 任何的 client 都可以使用 public key 对签名进行验证, 而 server 需要使用 private key 进行签名; 

这种 key pair 也是比较基本的非对称加密, 一般的使用 rsa 的方式生成即可 (ssh-keygen 也行)

```python
import rsa

# Generate an RSA key pair
(public_key, private_key) = rsa.newkeys(2048)

# Save the keys to files
with open("public_key.pem", "wb") as public_key_file:
    public_key_file.write(public_key.save_pkcs1())

with open("private_key.pem", "wb") as private_key_file:
    private_key_file.write(private_key.save_pkcs1())

print("RSA key pair generated and saved as public_key.pem and private_key.pem")
```

>   这里需要 python 环境 rsa, 不管是 pip 还是 conda 反正装上才能用

