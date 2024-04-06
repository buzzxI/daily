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

# spring security

## jwt token

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

这种 key pair 也是比较基本的非对称加密, 这里可以使用 openssl 即可生成对应的密钥对 (ssh-keygen 也行)

## security

其实说白了还是 filter 那一套, spring security 采用一种十分复杂的方式表示串联各个 filter, 进行认证并鉴权, 整体结构为:

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/03/11/20:49:26:spring_security_architecture.jpg)

1.   对于任意的 http request, spring security 会首先通过 filter 过滤, 试图通过 filter 对 request 进行 authentication/authorization, 上图所示的 Authentication Filter 一般是一个 filter chain

     比如 UsernamePasswordAuthenticationFilter, 就是从 request 中根据 username 和 password, 对请求进行授权

     在使用 jwt 认证的场景中, 经典行为就是先通过 username 和 password 进行认证, 并对认证的用户返回一个 jwt token 该 token 包含了用户的身份信息, 后续的请求需要携带该 token, 这样 server 才能对其进行认证和鉴权

2.   在 filter chain 中, 合适的 filter 会对 request 进行处理, 并在当前请求的上下文中保存一个 AuthenticationToken

3.   这里的 AuthenticationToken, 仅仅是对 request 中携带的身份信息的一个封装, 尚未进行实际的认证/鉴权

4.   spring security 为了功能解耦, 支持多种认证/鉴权方式, 默认的认证的行为被一个 Manager 进行管理, 对外界而言, 暴露一个 authenticate 方法, 在上一步得到的 AuthenticationToken, 会调用该接口进行认证/鉴权

     AuthenticationManager 是一个接口, 实际的 Manager 使用默认的 ProviderManager

5.   为了支持多种认证方式, 每种认证方式都通过 AuthenticationProvider 进行抽象, 比较常用的是 DaoAuthenticationProvider, 从名字也能看出来是和 dao 相关的认证方式, 表示需要对于 AuthenticationToken 和持久化的数据进行比较进行认证/鉴权

6.   UserDetailsService 本身也是一个接口, 定义了方法 loadUserByUsername, 表示根据用户名查找用户, AuthenticationProvider 需要依靠 UserDetailsService 根据用户名查找本地保存的 user

7.   UserDetailsService 会返回一个符合 spring security 鉴权格式的 UserDetails 对象, 后续在 AuthenticationProvider 中比较 request 中的身份信息和返回的 UserDetails 中的身份信息; 通常比较的是密码

8.   一系列 AuthenticationProvider 会将认证/鉴权的结果返回给 AuthenticationManager, 在 manager 处对认证的结果进行判断

9.   AuthenticationManager 返回的是一个被认证/鉴权后的 AuthenticationToken

10.   该 AuthenticationToken 会被保存在上下文中, 和请求一起进入 controller, 此后 spring security 完成了认证/鉴权

借助 spring security, 后续在 controller, service 中都可以访问到当前请求下的身份信息; 本质上 spring security 就是希望通过 filter 将 request 中的身份信息具化, 并保存在调用链的上下文中

## talk is cheap

>   本来想着一个鉴权能有多复杂, 但实际写的时候才发现 code is sooooooo expensive ...

整个例子使用了: spring-security (废话), jjwt 作为 jwt 操作的工具类, redis 作为 nosql 加速多次针对同一个用户的查找, mybatis-plus 作为持久化层的框架, mysql 作为数据库, 保存了一个用户表 ...

>   其实是想做 user <-> role <-> privilege 的多对多的映射关系的, 太麻烦了, 一个示例不需要这么麻烦

整个示例的框架如下

```shell
├─config		# 一些配置类, 比如 security, redis, jwt, mybatis
├─constant		# 用到的一些常量信息
├─controller	# 应该不用介绍了
├─dto			# dto 保存的是 controller 和前端交互的数据, request/response
├─entities		# 主要是 user 类
├─exception		# 自定义的异常类
├─filter		# jwt 示例的关键, 自定义了一个服从 spring security 规范的 filter
├─handler		# 处理各种异常的 handler
├─mapper		# UserMapper 
├─service		# 应该也不用介绍了
│  └─impl
└─util			# 工具类就是把 redis template 再封装了一下
```

### redis

>   先把边缘的配置说一下

其实就是封装了一下 RedisTemplate, 本例中 redis 不是关键, 所以还是按照常规, 配置了一个 String -> Object 类型的 RedisTemplate, 比较通用, 也比较常用吧

```java
package icu.buzz.security.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean("String2ObjectTemplate")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();

        template.setConnectionFactory(factory);

        // string as key
        StringRedisSerializer keySerializer = new StringRedisSerializer();
        template.setKeySerializer(keySerializer);
        template.setHashKeySerializer(keySerializer);

        // object as value
        ObjectMapper objectMapper = new ObjectMapper();
        // serialize all: getter/setter/fields, regardless of access modifier
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // serialize non-final class only
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
        Jackson2JsonRedisSerializer<Object> valueSerializer = new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        template.setValueSerializer(valueSerializer);
        template.setHashValueSerializer(valueSerializer);
        // make sure all configuration is set
        template.afterPropertiesSet();
        return template;
    }
}
```

config 类的作用是向 spring 中注入一个 RedisTemplate 作为 bean, 其实有这个 bean 就可以直接操作 key 和 value 了, 但还是使用了一个 RedisUtils 进行了封装

```java
package icu.buzz.security.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Component
public class RedisUtil {
    // evaluate the value of the default-expire property
    @Value("${spring.data.redis.default-expire}")
    private long defaultExpire;

    private final RedisTemplate<String, Object> template;

    @Autowired
    public RedisUtil(RedisTemplate<String, Object> template) {
        this.template = template;
    }

    public Object get(String key) {
        return template.opsForValue().get(key);
    }

    public void set(String key, Object value) {
        this.set(key, value, defaultExpire);
    }

    public void set(String key, Object value, long timeout) {
        template.opsForValue().set(key, value, timeout, TimeUnit.SECONDS);
    }

    public void delete(String key) {
        template.delete(key);
    }
}
```

就是为各个 key 添加了一个过期时间, 该时间可以通过配置文件进行配置, 这里默认的过期时间设置的比较长, 设置了 24 小时

```yaml
spring:
  data:
    redis:
      port: 6380
      host: localhost
      default-expire: 86400 # 60 * 60 * 24
```

>   有 docker 之后, 这些服务就随便起了, 我这里是把 redis 放在了 6380 端口上

### jwt

有关 jwt 的相关配置, 在最新的 jjwt 库中, 要求必须使用 private key 进行签名, 而使用 public key 进行校验, 因此这里还需要配置 key 的存放位置

```java
@ConfigurationProperties(prefix = "jwt")
public record JwtConfig(long expireTime, RsaKeys rsaConfig) {
    public record RsaKeys(RSAPublicKey publicKey, RSAPrivateKey privateKey) {
    }
}
```

server 颁发的 jwt 包含了过期时间信息, 同时这里借助 jdk 16 提供的 record 特性, 保存各个参数

```yaml
jwt:
  # god-damn it, spring does not support EL in yml file
  expire-time: 7200 # 60 * 60 * 2
  rsa-config:
    private-key: classpath:cert/private_key.pem
    public-key: classpath:cert/public_key.pem
```

>   spring 在 yaml 文件中并不支持 EL 表达式, 这意味着配置时间参数的时候需要手动计算出来 ...

这里默认将 key 保存在 resource 下的 cert 目录中了, 本例使用的 private-key 和 public-key 都是使用 openssl 生成的, 采用的是 rsa 算法

jwt 有关的操作本质上就是两种, 要么是颁发签名, 要么是验证签名

```java
package icu.buzz.security.service;

import icu.buzz.security.config.JwtConfig;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Date;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
public class JwtService {

    private JwtConfig jwtConfig;

    @Autowired
    public void setJwtConfig(JwtConfig jwtConfig) {
        this.jwtConfig = jwtConfig;
    }
	
    // 生成一个 jwt token
    public String generateToken(Authentication authentication) {
        Instant now = Instant.now();

        String scope = authentication
                .getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.joining(" "));

        return Jwts.builder()
                .issuer("self")
                .subject(authentication.getName())
                .claims(Map.of("scope", scope))
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plus(jwtConfig.expireTime(), ChronoUnit.SECONDS)))
                .signWith(jwtConfig.rsaConfig().privateKey())
                .compact();
    }
	
    // 检查 token 是否过期, 其实就是比较一下当前时间和 token 中的时间信息
    public boolean tokenExpire(String token) {
        return extractExpiration(token).before(new Date());
    }

    // 提取 jwt 中的用户信息 (也就是主体, subject)
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
	
    // 提取 jwt 中的颁发日期信息
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    // 提取 jwt 中的某个 claim, 具体的类型通过 Function 决定
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
	
    // 提取 jwt 中的各个 claim
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(jwtConfig.rsaConfig().publicKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

注意到生成 jwt token 的时候参数是一个 Authentication, 其实是 AuthenticationManager 的返回值

### login

>   前面的那些也就是凉菜, 先上点热菜

每个 login 请求都包含两部分, username 和 password, 这里也使用了对象进行封装

```java
package icu.buzz.security.dto;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class LoginRequest {
    private String username;
    private String password;
}
```

有关权限相关的操作都被抽象到了 AuthController 中

```java
package icu.buzz.security.controller;

import icu.buzz.security.constant.SecurityConstant;
import icu.buzz.security.dto.LoginRequest;
import icu.buzz.security.service.AuthService;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/login")
    public ResponseEntity<Void> login(@RequestBody LoginRequest request) {
        String jwt = authService.authenticate(request);
        HttpHeaders httpHeaders = new HttpHeaders();
        httpHeaders.set(SecurityConstant.TOKEN_HEADER, SecurityConstant.TOKEN_PREFIX + jwt);
        return new ResponseEntity<>(httpHeaders, HttpStatus.OK);
    }
}
```

基本上所有的逻辑都抽象放在了 service 中了, controller 就只是把 token 放在 response header 中了

```java
public class SecurityConstant {
    public static final String TOKEN_HEADER = "Authorization";
    public static final String TOKEN_PREFIX = "Bearer ";
}
```

jwt 保存在 http response header 中的 Authentication 字段中, 并且保留了一个前缀 Bearer

```java
@Service
public class AuthServiceImpl implements AuthService {
    
    @Override
    public String authenticate(LoginRequest request) {
        try {
            Authentication authenticate = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            request.getUsername(),
                            request.getPassword()));
            // user may not be available: disabled user (banned), locked (danger), expired (temp user)
            if (!authenticate.isAuthenticated()) {
                throw new UserNotAvailableException(Map.of(ExceptionConstant.USER_NOT_AVAILABLE, request.getUsername()));
            }
            return jwtService.generateToken(authenticate);
        } catch (BadCredentialsException e) {
            throw new PasswordMismatchException(Map.of(ExceptionConstant.BAD_CREDENTIALS, request.getUsername()));
        }
    }
}
```

方法 authenticate 首先根据 request 中包含了 username 和 password 信息生成一个未认证/授权的 AuthenticationToken, 并将该 token 交给 AuthenticationManager 进行授权/认证, 最终返回一个被认证的 AuthenticationToken, 该 token 会交给 jwt service 生成表示了用户身份信息的 jwt

注意到校验 AuthenticationToken 的时候, 是可能不通过的, 在 spring security 中通过 BadCredentialException 体现, 为了对异常进行全面的掌控, 这里将其转化为一个自定义的 PasswordMismatchException, 这个异常可以用来指示用户名和密码不匹配, 这个异常会被自定义的全局异常处理类捕获

在本例中 AuthenticationManager 是作为一个 bean 被注入当前 service 中的, 具体的有关 AuthenticationManager 的配置放在了配置类 AuthConfig 中

```java
package icu.buzz.security.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.List;

@Configuration
public class AuthConfig {

    private UserDetailsService userDetailsService;

    @Autowired
    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    /**
	 * just as the name said, encode the password (with salt dynamically)
	 */
	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}

    /**
	 * query user info by UserDetailService
	 * retrieve UserDetails by username -> UsernameNotFoundException
	 * check user status -> some other exception
	 * compare password -> BadCredentialsException
	 */
	@Bean
	public DaoAuthenticationProvider daoAuthenticationProvider(PasswordEncoder passwordEncoder) {
		DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
		provider.setPasswordEncoder(passwordEncoder);
		provider.setUserDetailsService(this.userDetailsService);
		return provider;
	}

	/**
	 * control the way to authenticate the user
	 */
	@Bean
	public AuthenticationManager authenticationManager(DaoAuthenticationProvider daoAuthenticationProvider) {
		return new ProviderManager(List.of(daoAuthenticationProvider));
	}
}
```

AuthenticationManager 需要若干个 AuthenticationProvider, 这里通过一个 List 存储 (其实也就是一个 DaoAuthenticationProvider); 每个 AuthenticationProvider 都需要配置好 UserDetialsService 的实现类, 用来获取实际的用户身份信息; spring security 也支持对密码进行加密, 将密码以明文的形式保存的确不妥, 因此也配置了一个 password encoder

### UserDetailsService

这部分其实主要是从持久化层根据 username 获取用户的操作

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    private final RedisUtil redisUtil;
    private final UserMapper userMapper;

    @Autowired
    public UserDetailsServiceImpl(RedisUtil redisUtil, UserMapper userMapper) {
        this.redisUtil = redisUtil;
        this.userMapper = userMapper;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = (User) redisUtil.get(username);
        if (user == null) {
            // query by mybatis-plus
            List<User> users = userMapper.selectList(new QueryWrapper<User>().lambda().eq(User::getUsername, username));
            if (users.isEmpty()) {
                throw new UsernameNotFoundException(Map.of(ExceptionConstant.USERNAME_NOT_FOUND, username));
            }
            if (users.size() > 1) {
                // multiple users found, that should be server internal error
                throw new MultiUserFoundException(Map.of(ExceptionConstant.MULTIPLE_USER_FOUND, username));
            }
            user = users.get(0);
            redisUtil.set(user.getUsername(), user);
        }
        return new JwtUser(user);
    }
}
```

首先会从 redis 中根据用户名查找一个 User, 如果找不到的话才会借助 mybatis 到数据库中查询, 注意到这个方法最终需要返回的是一个 UserDetails 对象, 因此实际保存的类型 User 需要封装一下, 在本实例中, 使用 JwtUser 继承了 UserDetails 保存了用户信息

```java
package icu.buzz.security.entities;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.experimental.Accessors;

@Data
@AllArgsConstructor
@NoArgsConstructor
// enable chain style (lombok)
@Accessors(chain = true)
// associate entity with table user
@TableName("`users`")
public class User  {
    // primary key, auto generate by mybatis-plus (uuid)
    @TableId(value = "uid", type = IdType.ASSIGN_UUID)
    private String uid;
    private String username;
    private String password;
    private Boolean enabled;
    private Role role;
}
```

在实际的数据库中, 保存的信息和 UserDetails 中的信息并不完全对应, 这里采用 uuid 作为 user 表的主键, 同时保存了 username, password(加密后); 同时针对用户的有效性, 使用字段 enabled 表示; 为了表示用户的身份信息, 使用枚举类 Role 封装

```java
package icu.buzz.security.entities;

import com.baomidou.mybatisplus.annotation.EnumValue;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

@Getter
public enum Role {
    ADMIN(1, Set.of(Privilege.READ, Privilege.WRITE, Privilege.UPDATE, Privilege.DELETE), "admin"),
    USER(2, Set.of(Privilege.READ, Privilege.WRITE), "user"),
    ;

    @EnumValue
    private final int id;
    private final Set<Privilege> privileges;
    private final String desc;

    Role(int id, Set<Privilege> privileges, String desc) {
        this.id = id;
        this.privileges = privileges;
        this.desc = desc;
    }

    public List<GrantedAuthority> getAuthority() {
        return privileges
                .stream()
                .map((p) -> new SimpleGrantedAuthority(p.name()))
                .collect(Collectors.toList());
    }
}
```

注意到每种角色还包含了权限信息, 这一点其实符合 RBAC 模型 (role-based access control), 每个用户有多种身份, 每种身份又有多种权限

```java
package icu.buzz.security.entities;

import com.baomidou.mybatisplus.annotation.EnumValue;
import lombok.Getter;

@Getter
public enum Privilege {
    READ(1, "read"),
    WRITE(2, "write"),
    DELETE(3, "delete"),
    UPDATE(4, "update");

    @EnumValue
    private final int id;
    private final String desc;

    Privilege(int id, String desc) {
        this.id = id;
        this.desc = desc;
    }
}
```

>   从注解 @EnumValue 也能看出来, 当时设计的时候是想着把各种关系通过多表的形式保存在数据库中的, 但毕竟 role 和 privilege 的映射关系相对比较固定, 并且也只是想着让 role 以一种包含的关系呈现, 因此也没有设计 user 和 role 的多表关系

类型 Role 还提供了一个莫名其妙的方法 getAuthority 这个其实是 UserDetails 的要求, 具体的引用可以看 JwtUser

```java
package icu.buzz.security.entities;

import lombok.Data;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

@Data
public class JwtUser extends User implements UserDetails {
    public JwtUser() {
    }

    public JwtUser(User user) {
        super(user.getUid(), user.getUsername(), user.getPassword(), user.getEnabled(), user.getRole());
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Role role = super.getRole();
        List<GrantedAuthority> authority = role.getAuthority();
        authority.add((GrantedAuthority) () -> "ROLE_" + role.name());
        return authority;
    }

    @Override
    public String getPassword() {
        return super.getPassword();
    }

    @Override
    public String getUsername() {
        return super.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return super.getEnabled();
    }
}
```

其实和 User 没什么区别, 主要在于实现了接口 UserDetails, 主要是实现了方法 getAuthorities, 返回权限信息

### register

除了简单的登录操作之外, 还提供了注册功能, 因为也是和权限控制相关的, 因此将该功能放在了 AuthController 中

```java
/**
 * when user sign up, an email is needed, but for now, just use LoginRequest to simulate the sign-up process
 */
@PostMapping("/sign-up")
public ResponseEntity<Void> signUp(@RequestBody LoginRequest request) {
    User user = authService.signUp(request);
    return ResponseEntity.ok().build();
}
```

一般情况下用户注册的时候都需要填写一堆信息, 这里就一切从简, 直接使用 LoginRequest 封装, 现在的用户注册输入用户名和密码就好了

```java
@Service
public class AuthServiceImpl implements AuthService {

    @Override
    public User signUp(LoginRequest request) {
        if (userDetailsService.userPresent(request.getUsername())) {
            throw new UsernameAlreadyExistException(Map.of(ExceptionConstant.USERNAME_ALREADY_EXIST, request.getUsername()));
        }
        return userDetailsService.saveUser(request.getUsername(), passwordEncoder.encode(request.getPassword()));
    }
}
```

注册之前首先需要确定当前用户尚未被注册过, 首先需要从 UserDetailsServiceImpl 中查找当前用户名下用户的注册信息, 如果当前用户已经注册过了, 会抛出自定义异常 UsernameAlreadyExistException

否则, 只要当前用户没有注册过, 就可以将该用户名和密码保存在数据库中 -> 注意这里在保存密码的时候需要使用注入的 password encoder 对密码编码 (在用户的登录的时候不需要手动编码, spring security 会在校验的时候编码, 但注册页并没有直接被 spring security 管理, 这里需要手动编码)

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    private final RedisUtil redisUtil;
    private final UserMapper userMapper;

    public boolean userPresent(String username) {
        Object jwtUser = redisUtil.get(username);
        if (jwtUser != null) {
            return true;
        }
        List<User> users = userMapper.selectList(new QueryWrapper<User>().lambda().eq(User::getUsername, username));
        return !users.isEmpty();
    }

    public User saveUser(String username, String password) {
        User user = new User();
        user
                .setUsername(username)
                .setPassword(password)
                .setRole(Role.USER)
                .setEnabled(true);
        userMapper.insert(user);
        return user;
    }
}
```

因为本例使用了 redis 加速用户查找, 以 username 为 key, 以查询到的 User 为 value; 因此这里在检查用户是否存在的时候不会无脑查询数据库, 先到 no sql 里面查查看

注册用户的时候仅仅提供了用户名和密码, 因此这里保存用户的时候一切其他的字段都被填写为默认值

### logout

有登录就有登出

```java
@PostMapping("/logout")
public ResponseEntity<Void> logout() {
    authService.logout();
    return ResponseEntity.ok().build();
}
```

反正一切权限相关的信息都交给 AuthService 处理了

```java
@Override
public void logout() {
    UserDetails currentUser = contextUtil.getCurrentUser();
    userDetailsService.unloadUserByUsername(currentUser.getUsername());
}
```

因为只要用户登录了, 那么在当前调用链的上下文中, spring security 就会保存用户信息 -> UserDetails; 所以 logout 请求时是需要携带 jwt 信息的, 未携带 jwt 的 logout 请求都会被 spring security 拦截 (显然注册请求是不需要携带 jwt 信息的)

UserDetailsService 接口需要实现方法 loadUserByUsername, 这个是 spring security 根据用户名查询用户信息的方法, 从而完成用户登录; 而现在到了登出接口, 这里就自定义了方法 unloadUserByUsername, 表示根据用户名完成登出

```java
package icu.buzz.security.util;

import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

@Component
public class ContextUtil {

    public UserDetails getCurrentUser() {
        return (UserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    }
}
```

ContextUtil 其实没有什么特别的, 就是调一下 spring security 默认提供的方法 ...

>   后面会说一下这个调用链

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    /**
     * remove User by username
     */
    public void unloadUserByUsername(String username) {
        redisUtil.delete(username);
    }
}
```

在登录的最后一个阶段, 主动将 username -> User pair 保存在 redis 中, 这里的登出也就是把这个 pair 删除掉

### security config

>   大的要来了

讲道理, 之前的 AuthConfig 也算是 security config 的一部分, 配置了 AuthenticationManager, AuthenticationProvider, PasswordEncoder

而这里的 security 更进一步, 进行了具体的权限控制; 而且在 spring security 6 之后, 这个配置类和之前的配置已经很不一样了, 官网上可以查到 ...

```java
package icu.buzz.security.config;

import icu.buzz.security.constant.SecurityConstant;
import icu.buzz.security.entities.Role;
import icu.buzz.security.filter.JwtAuthenticationFilter;
import icu.buzz.security.handler.CustomAccessDeniedHandler;
import icu.buzz.security.handler.CustomAuthenticationEntryPoint;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

// this configuration will enable default SecurityFilterChain
@EnableWebSecurity
@Configuration
public class SecurityConfig {
	private JwtAuthenticationFilter jwtAuthenticationFilter;

	private CustomAuthenticationEntryPoint customAuthenticationEntryPoint;
	private CustomAccessDeniedHandler customAccessDeniedHandler;

	@Autowired
	public void setJwtAuthenticationFilter(JwtAuthenticationFilter jwtAuthenticationFilter) {
		this.jwtAuthenticationFilter = jwtAuthenticationFilter;
	}

	@Autowired
	public void setCustomAuthenticationEntryPoint(CustomAuthenticationEntryPoint customAuthenticationEntryPoint) {
		this.customAuthenticationEntryPoint = customAuthenticationEntryPoint;
	}

	@Autowired
	public void setCustomAccessDeniedHandler(CustomAccessDeniedHandler customAccessDeniedHandler) {
		this.customAccessDeniedHandler = customAccessDeniedHandler;
	}

	@Bean
    public SecurityFilterChain filterChain(HttpSecurity http, AuthenticationManager authenticationManager) throws Exception {
		http
				// any request must be authenticated
				.authorizeHttpRequests((authorize) -> authorize
						// permit request to `login` and `sign up`
						.requestMatchers(HttpMethod.POST, SecurityConstant.SYSTEM_WHITELIST).permitAll()
						// only users with admin can access `/admin/**`
						.requestMatchers(SecurityConstant.ADMIN_RESOURCE).hasRole(Role.ADMIN.name())
						// all users signing up can access `/user/**`
						.requestMatchers(SecurityConstant.USER_RESOURCE).hasAnyRole(Role.USER.name(), Role.ADMIN.name())
						// anonymous users can access `/public/**`
						.requestMatchers(SecurityConstant.PUBLIC_RESOURCE).permitAll()
						.anyRequest().authenticated())
				// in a jwt based system, csrf protection is not needed -> (two stage token is recommended -> refresh token and in-memory jwt token)
				.csrf(AbstractHttpConfigurer::disable)
				// jwt filter must be before UsernamePasswordAuthenticationFilter
				// server will check the jwt token first, then use username&password second
				// all request need to be authenticated, if jwt filter hits, username&password filter will not be executed
				.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
				.authenticationManager(authenticationManager)
				.httpBasic(Customizer.withDefaults())
				// the server never creates a session
				.sessionManagement((session) -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
				.exceptionHandling((exception) -> exception
						.authenticationEntryPoint(customAuthenticationEntryPoint)
						.accessDeniedHandler(customAccessDeniedHandler)
				);
		// most time BearerTokenAuthenticationEntryPoint and BearerTokenAccessDeniedHandler are enough
//				.exceptionHandling((exception) -> exception
//						.authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
//						.accessDeniedHandler(new BearerTokenAccessDeniedHandler())
//				);

		return http.build();
	}

}
```

这里 SecurityConfig 其实就是向 spring 注入了一个 SecurityFilterChain 的 bean, 这个 filter chain 是 spring security 实现请求过滤的关键; 在 spring security 官方的建议里, 推荐使用 lambda 的方式进行配置

首先配置了拦截/放行的路径, 默认放行登录和注册页面 (仅 post 请求); 所有路径带有 /admin 的请求都需要当前用户至少具有 admin role, 所有路径带有 /user 的请求都需要当前用户至少需要 user role (从设计上 admin role 包含了 user role)

>   其实 /admin 和 /user 下啥也没有, 这里就是给出两个示例表示 spring security 对于权限的控制

然后关闭了 csrf 攻击的防御, 因为本例使用的 jwt 信息并不会保存在 cookie 中; 然后配置了自定义的 Filter, JwtAuthenticationFilter, 这个过滤器会尝试根据 http request 中的 jwt 获取用户信息, 并保存在 spring security 的 context 中

>   从认证行为上, JwtAuthenticationFilter 和 UsernamePasswordAuthenticationFilter 是类似的, 一个是根据 jwt 获取 UserDetails, 一个是根据 username 和 password 获取 UserDetails

然后为该调用链配置 AuthenticationManager (在之前的 AuthConfig 中注入的 bean); 因为 jwt 是无状态的, 因此 server 并不需要维护 session, 所以这里也将 session 关闭了

最后就是配置了异常处理类, 这里主要配置了两个异常处理类, 一个是 AuthenticationEntryPoint 类型, 表示用户未通过授权; 另一个是 AccessDeniedHandler 表示该用户通过了认证, 但认证信息中用户的权限不足以访问其需要访问的资源

>   很多时候这两个 handler 并没有想象中的那么好用, 所以有些异常我都通过转发的形式主动让全局的异常处理器处理了 ...

### jwt filter

该 filter 的任务其实很简单, 就是解析 jwt token, 然后根据解析的 jwt token 获取用户信息, 这里的 filter 继承了 BasicAuthenticationFilter, 就像它在源码中描述的那样, 该 filter 的主要目的是解析 http header 中的 Authentication 字段

```java
package icu.buzz.security.filter;

import icu.buzz.security.constant.SecurityConstant;
import icu.buzz.security.service.JwtService;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.www.BasicAuthenticationFilter;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerExceptionResolver;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends BasicAuthenticationFilter {

    private JwtService jwtService;
    private UserDetailsService userDetailsService;

    private HandlerExceptionResolver handlerExceptionResolver;

    @Autowired
    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    @Autowired
    public void setJwtService(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Autowired
    public void setUserDetailsService(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    @Autowired
    @Qualifier("handlerExceptionResolver")
    public void setHandlerExceptionResolver(HandlerExceptionResolver handlerExceptionResolver) {
        this.handlerExceptionResolver = handlerExceptionResolver;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String token = request.getHeader(SecurityConstant.TOKEN_HEADER);

        if (token == null || !token.startsWith(SecurityConstant.TOKEN_PREFIX)) {
            SecurityContextHolder.clearContext();
            chain.doFilter(request, response);
            return;
        }

        // check jwt token
        token = token.replace(SecurityConstant.TOKEN_PREFIX, "");
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            try {
                if (!jwtService.tokenExpire(token)) {
                    String username = jwtService.extractUsername(token);
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                    // current context need an authentication token, just new an instance
                    // set UserDetails as principal, token as credentials
                    UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(userDetails, token, userDetails.getAuthorities());
                    SecurityContextHolder.getContext().setAuthentication(authenticationToken);
                }
            } catch (JwtException e) {
                handlerExceptionResolver.resolveException(request, response, null, e);
                return;
            }
        }
        chain.doFilter(request, response);
    }
}

```

filter 首先会尝试获取 http header 中的 Authentication 字段, 要注意的是, 本例中 jwt token 都是以 Bearer 开头的, 因此这里还对前缀进行了检测

要注意的是, 请求首先会进入 filter 才会进到 controller 中, 在 filter 中抛出的异常不会被 GlobalExceptionHandler 捕获, 因此这里需要使用 HandlerExceptionResolver 强制将异常交给 GlobalExceptionHandler 处理

>   GlobalExceptionHandler 通过 @RestControllAdvice 声明, 处理的是在 controller 调用链中出现的异常

filter 首先检查 jwt 的是否过期, 然后从 jwt 中提取用户名, 并通过 UserDetailsService 获取该用户, 并讲查询到的 UserDetails 保存在 spring security 的上下文中

>   这里其实存在隐患, 因为 loadUserByUsername 其实是有可能抛出 runtime exception 的, 分别是查询不到用户, 和查询到了多个用户, 总之是 server 内部的问题, 但如果不处理这个异常的话, server 应该会直接挂掉

注意到这里保存的 AuthenticationToken 和之前在处理 login 请求时创建的类型相同, 只不过在 login 中 token 还需要进一步认证, 调用 authentication manager 进一步处理, 而在 jwt filter 中, 声明的就是一个已经授权好的 AuthenticationToken 了, 直接保存在 security 的上下文中即可

这里的 AuthenticationToken 在创建时, 需要传入三个参数: principal, credential, authorities, 分别对应了实际传入的参数, userDetails, token, userDetails.getAuthorities(); 正是因此, 之前在 ContextUtil 中, 需要获取当前用户的时候, 获取的是 authentication 的 principal

### expcetion and handler

在本例中, 所有的 exception 都继承了类 BaseException

```java
package icu.buzz.security.exception;

import icu.buzz.security.entities.ErrorCode;
import lombok.Getter;

import java.util.Map;

@Getter
public class BaseException extends RuntimeException {
    private final ErrorCode errorCode;
    private final transient Map<String, Object> map;

    public BaseException(ErrorCode errorCode, Map<String, Object> map) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
        this.map = map;
    }
}
```

本质上也是一个 RuntimeException, 在其基础上定义了类型 ErrorCode 保存错误信息, 并使用一个 map 表示具体出错的内容 (可选部分)

```java
package icu.buzz.security.entities;

import lombok.Getter;
import org.springframework.http.HttpStatus;

@Getter
public enum ErrorCode {
    USER_NAME_ALREADY_EXIST(1001, HttpStatus.BAD_REQUEST,  "user already exist"),
    USER_NAME_NOT_FOUND(1002, HttpStatus.NOT_FOUND , "user not found"),
    INVALID_JWT_TOKEN(1003, HttpStatus.UNAUTHORIZED, "jwt token illegal"),
    MULTIPLE_USER_FOUND(1004, HttpStatus.INTERNAL_SERVER_ERROR, "multiple user found"),
    PASSWORD_MISMATCH(1005, HttpStatus.BAD_REQUEST, "password mismatch"),
    USER_NOT_AVAILABLE(1006, HttpStatus.FORBIDDEN, "user not available"),
    EXPIRED_JWT_TOKEN(1007, HttpStatus.UNAUTHORIZED, "jwt token expired"),
    ;

    private final int code;
    private final HttpStatus status;
    private final String message;

    ErrorCode(int code, HttpStatus status, String message) {
        this.code = code;
        this.status = status;
        this.message = message;
    }
}
```

ErrorCode 是一个枚举类型, 封装了异常消息 message, 以及出现该异常时, 需要设置的 http status code; 并再次基础上, 自定义了 error code, 用来指示前端具体的错误信息

这里一共定义了 7 种 ErrorCode, 对应了自定义的 7 种异常, 具体的每种异常的写法都是类似的, 就不多赘述了

```java
package icu.buzz.security.handler;

import icu.buzz.security.dto.ErrorResponse;
import icu.buzz.security.exception.*;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * handle exception: current username has already in use
     */
    @ExceptionHandler(UsernameAlreadyExistException.class)
    public ResponseEntity<ErrorResponse> handleUsernameAlreadyExist(UsernameAlreadyExistException e, HttpServletRequest request) {
        return simpleErrorHandling(e, request.getRequestURI());
    }

    /**
     * handle exception: username not found
     */
    @ExceptionHandler(UsernameNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUsernameNotFound(UsernameNotFoundException e, HttpServletRequest request) {
        return simpleErrorHandling(e, request.getRequestURI());
    }

    @ExceptionHandler(PasswordMismatchException.class)
    public ResponseEntity<ErrorResponse> handlePasswordMismatch(PasswordMismatchException e, HttpServletRequest request) {
        return simpleErrorHandling(e, request.getRequestURI());
    }

    /**
     * handle exception: multiple users found (internal exception)
     */
    @ExceptionHandler(MultiUserFoundException.class)
    public ResponseEntity<ErrorResponse> handleMultiUserFound(MultiUserFoundException e, HttpServletRequest request) {
        return simpleErrorHandling(e, request.getRequestURI());
    }
    
    /**
     * handle exception: user not available (disabled, locked, expired)
     */
    @ExceptionHandler(UserNotAvailableException.class)
    public ResponseEntity<ErrorResponse> handleUserNotAvailable(UserNotAvailableException e, HttpServletRequest request) {
        return simpleErrorHandling(e, request.getRequestURI());
    }
    
    @ExceptionHandler(JwtException.class)
    public ResponseEntity<ErrorResponse> handleJwtException(JwtException e, HttpServletRequest request) {
        return simpleErrorHandling(new InvalidJwtTokenException(Map.of("invalid jwt token", e.getMessage())), request.getRequestURI());
    }

    /**
     * simple error handling
     * construct an error response from exception
     */
    private ResponseEntity<ErrorResponse> simpleErrorHandling(BaseException e, String uri) {
        ErrorResponse response = new ErrorResponse(e.getErrorCode(), uri, e.getMap());
        return ResponseEntity.status(e.getErrorCode().getStatus()).body(response);
    }
}

```

简单来说, 出现了异常后, 就返回一个 ErrorResponse 对象, 该对象通过对 ErrorCode 和请求 uri 封装得到

## one more thing

上面仅仅是基础的 spring security 和 jwt 的简单应用, 实际的业务开发需求可能更加复杂

### limit login attempts

一个很简单的需求, 如果一个账户在一段时间内登录此时过多, 则会被判定为风险账户, 对账户进行锁定, 一段时间内禁止风险账户登录, 并在一段时间后自动解除锁定

显然这个需求需要记录一个账户在某段时间内的登录次数, 这里在 redis 中通过 key-value pair 维护, 以 <用户名-ip> 为 key, 并以 <失败次数> 为 value, 每次失败的尝试过后自增 value

spring security 提供了很丰富的接口, 其中就包括了监听 authentication 成功/失败的事件, 因此这里可以直接在 authentication 失败的事件处理函数中对登录失败的次数进行统计

>   在 spring 中事件机制的核心是: event, listener, publisher
>
>   其中 event 是事件类型, 默认的自定义的事件需要实现接口 ApplicationEvent
>
>   listener 中定义了事件处理方法, 监听特定的事件需要实现接口 ApplicationListener\<T\>, 其中 T 表示自定义的事件类型
>
>   publisher 用来触发事件, 一般而言自定义的 publisher 中需要保留一个 ApplicationEventPublisher 的引用, 通过调用 publisher 的 publishEvent 方法进行事件触发 (从而被 listener 处理)
>
>   一般而言 listener 和 publisher 需要通过 bean 的形式被 spring container 管理, 其中用户程序可以调用 publish 的 publish event 方法进行事件触发
>
>   举例来说, 一组可能的 event, listener, publisher 具有如下形式:
>
>   ```java
>   // CustomEvent.java
>   
>   import org.springframework.context.ApplicationEvent;
>   
>   public class CustomEvent extends ApplicationEvent {
>       private String message;
>   
>       public CustomEvent(Object source, String message) {
>           super(source);
>           this.message = message;
>       }
>   
>       public String getMessage() {
>           return message;
>       }
>   }
>   ```
>
>   ```java
>   // CustomListener.java
>   
>   import org.springframework.context.ApplicationListener;
>   import org.springframework.stereotype.Component;
>   
>   @Component
>   public class CustomEventListener implements ApplicationListener<CustomEvent> {
>   
>       @Override
>       public void onApplicationEvent(CustomEvent event) {
>           System.out.println("Received custom event - " + event.getMessage());
>       }
>   }
>   ```
>
>   ```java
>   // CustomPublisher.java
>   
>   import org.springframework.context.ApplicationEventPublisher;
>   import org.springframework.stereotype.Component;
>   
>   @Component
>   public class CustomEventPublisher {
>       private final ApplicationEventPublisher publisher;
>   
>       public CustomEventPublisher(ApplicationEventPublisher publisher) {
>           this.publisher = publisher;
>       }
>   
>       public void publishCustomEvent(String message) {
>           CustomEvent event = new CustomEvent(this, message);
>           publisher.publishEvent(event);
>       }
>   }
>   ```

默认情况下 spring security 配置了 DefaultAuthenticationEventPublisher, 但是由于在 jwt 的示例中对 AuthenticationManager 重新进行了配置, 因此这里需要主动为 AuthenticationManager 配置 EventPublisher

```java
// AuthConfig.java

/**
 * control the way to authenticate the user
 */
@Bean
public AuthenticationManager authenticationManager(DaoAuthenticationProvider daoAuthenticationProvider, AuthenticationEventPublisher authenticationEventPublisher) {
    ProviderManager authenticationManager = new ProviderManager(List.of(daoAuthenticationProvider));
    // 为自定义的 AuthenticationManager 配置默认的 EventPublisher
    authenticationManager.setAuthenticationEventPublisher(authenticationEventPublisher);
    return authenticationManager;
}
```

这样 publisher 就配好了, 然后就是 authentication event 了, 在 spring security 中, 不仅可以通过继承接口实现 listener 的配置, 还可以通过注解 @EventListener

```java
// AuthenticationEventListener

@Component
public class AuthenticationEventListener {

    @EventListener
    public void onSuccess (AuthenticationSuccessEvent event) {
        
    }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
      	
    }
}
```

剩下的就是在 onFailure 中实现账户锁定的逻辑了, 基本上这里就是通过 redis 记录用户失败的次数, 在次数超出限制后, 直接锁定用户; 在 spring security 的 UserDetails 中, 本身就需要实现方法 isAccountNonLocked, 因此这里其实就是在 JwtUser 封装的属性中额外添加一个属性, locked 表示当前用户是否被锁定

但要注意的是, 这里的锁定会在一段事件后自动解除, 因此这里并没有使用一个 boolean 值表示, 反而使用了一个类型为 LocalDateTime 的字段 unlockedTime 表示账户解锁时间, 只要当前时间已经超过了 unlockedTime, 则当前账户就是可以正常访问的; 在锁定用户的时候直接将该值取为当前时间 + 锁定时间即可

之前在 UserDetailsServiceImpl 中实现方法 loadUserByUsername 的时候将查询得到的 User 直接保存在了 redis 中, 这样后续的查询就不需要经过数据库了, 直接查询 redis 即可

由于 User 本质上等价于 UserPo (持久化对象), 而 lock user 的行为本身是在内存中实现的, 没有必要将 unlockedTime 保存在 User 中; 但是将该字段保存在 JwtUser 中也并不合适, 因为实际保存在 redis 中的是 User 对象

>   不要想着用 JwtUser 替换 User 保存在 redis 中, JwtUser 的 authority 实在是太复杂了, json 序列化/反序列化总是出现问题

因此这里使用了一种折中的思路: (add a layer of indirection), 定义类型 UserBo, 其封装了对象 UserPo, 并在其基础上定义了字段 unlockedTime; 最终 JwtUser 中传入的对象变成了 UserBo, redis 中保存的也变成了 UserBo

```java
// UserBo.java

package icu.buzz.security.entities;

import lombok.Getter;
import lombok.Setter;

import java.time.LocalDateTime;

@Getter
@Setter
public class UserBo extends User {
    private LocalDateTime unlockedTime;

    public UserBo() {
    }

    public UserBo(User user) {
        super(user.getUid(), user.getUsername(), user.getPassword(), user.getEnabled(), user.getRole());
        this.unlockedTime = LocalDateTime.now();
    }
}
```

在 loadUserByUsername 方法中, 会向 redis 中添加一个 UserBo 对象, 并返回一个封装后的 JwtUser 对象

```java
// UserDetailsServiceImpl.java

@Override
public UserDetails loadUserByUsername(String username) {
    UserBo user = (UserBo) redisUtil.get(username);
    if (user == null) {
        // query by mybatis-plus
        List<User> users = userMapper.selectList(new QueryWrapper<User>().lambda().eq(User::getUsername, username));
        if (users.isEmpty()) {
            throw new UsernameNotFoundException(Map.of(ExceptionConstant.USERNAME_NOT_FOUND, username));
        }
        if (users.size() > 1) {
            // multiple users found, that should be server internal error
            throw new MultiUserFoundException(Map.of(ExceptionConstant.MULTIPLE_USER_FOUND, username));
        }
        user = new UserBo((users.get(0)));
        redisUtil.set(user.getUsername(), user);
    }
    return new JwtUser(user);
}
```

由于 redis template 配置了 value 以 json 的形式保存, 这里使用 spring boot 默认的 jackson 进行 json 序列化和反序列化, 而默认情况下 jackson 是不支持 jdk 8 时间类的序列化/反序列化, 这里需要额外的依赖

```xml
<!--pom.xml-->

<!--dependency for serialize json for LocalDataTime in jdk 8-->
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

同时在 ObjectMapper 中配置时间类的 module

```java
package icu.buzz.security.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.JsonDeserializer;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

@Configuration
public class RedisConfig {

    @Bean("String2ObjectTemplate")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();

        template.setConnectionFactory(factory);

        // string as key
        StringRedisSerializer keySerializer = new StringRedisSerializer();
        template.setKeySerializer(keySerializer);
        template.setHashKeySerializer(keySerializer);

        // object mapper cannot be injected as bean -> default mapper will be covered
        ObjectMapper objectMapper = new ObjectMapper();

        // add time module for java.time.* serialization
        objectMapper.registerModule(new JavaTimeModule());
        // serialize all: getter/setter/fields, regardless of access modifier
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // serialize non-final class only
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);

        // object as value
        Jackson2JsonRedisSerializer<Object> valueSerializer = new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        template.setValueSerializer(valueSerializer);
        template.setHashValueSerializer(valueSerializer);
        // make sure all configuration is set
        template.afterPropertiesSet();
        return template;
    }
}
```

此后 JwtUser 中的 isAccountNonLocked 方法会根据 userBo 中的 lock time 和当前时间的先后返回, 而不是直接返回一个 true

```java
// JwtUser.java

@Override
public boolean isAccountNonLocked() {
    return LocalDateTime.now().isAfter(user.getUnlockedTime());
}
```

对于限制用户登录的需求, 需要三个参数进行限制:

*   监控窗口: 对用户失败登录次数进行统计的窗口大小
*   失败登录次数: 在窗口内登录失败多少次后进行用户锁定
*   锁定时间: 触发锁定条件后, 一次性锁定用户的时间大小

将这三个参数放在配置文件中进行配置, 并另外使用一个 record 进行记录:

```java
package icu.buzz.security.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "buzz.security")
public record CommonConfig(LoginConfig loginConfig) {
    public record LoginConfig(int loginCount, int loginInterval, int lockTime) {
    }
}

```

剩下的就是完成两个事件处理函数了, 要注意的是 AuthenticationEventListener 中处理的两种事件 AuthenticationSuccessEvent 和 AbstractAuthenticationFailureEvent 内部都封装了一个 Authentication  对象, 其中 AuthenticationSuccessEvent 中包含的 Authentication 对象是一个经过了认证的对象, 在本环境中是一个 JwtUser 对象;  AbstractAuthenticationFailureEvent 中包含的 Authentication 对象是一个未经过认证的对象, 在本环境中就是原始的 username (字符串), 注意这两者 principal 的区别

>   要说明的是, spring security 的 DaoAuthenticaionProvider 在完成认证后会将 Credential 删除

因为 spring security 需要先调用 loadUserByUsername, 才进行 username, password 的比较, 因此就算比较认证失败, 那么 redis 中也一定保存了当前用户名的 UserBo 对象

```java
// AuthenticationEventListener.java

@EventListener
public void onFailure(AbstractAuthenticationFailureEvent event) {
    if (event.getException() instanceof BadCredentialsException) {
        String principal = event.getAuthentication().getPrincipal().toString();
    	UserBo user = (UserBo) redisUtil.get(principal);
        String userKey = constructUserKey(principal);
        Integer attemptCount = (Integer) redisUtil.get(userKey);
        if (attemptCount != null && attemptCount == commonConfig.loginConfig().loginCount()) {
            // ban user for a period of time
            user.setUnlockedTime(LocalDateTime.now().plusSeconds(commonConfig.loginConfig().lockTime()));
            redisUtil.set(principal, user);
        } else {
            if (attemptCount == null) attemptCount = 1;
            else attemptCount ++;
            redisUtil.set(userKey, attemptCount, commonConfig.loginConfig().loginInterval());
        }
    }
}

private String constructUserKey(String username) {
    return username + "_" + ipUtil.getIpAddr();
}
```

注意到导致用户认证失败可能有很多原因, 这里的计数仅仅针对于 username 和 password 不匹配的情况, 因此需要先判断当前异常的类型, 只有 BadCreditialsException 才需要这样处理

尽管还是使用 redis 进行失败登录次数进行计数, 但并没有将次数保存在 UserBo 中, 而是额外使用了一个和 ip 相关的 key, 这里使用 ipUtil 进行了 ip 地址的获取

当失败次数达到上限后, 会更新 UserBo 的 unlockedTime, 将其锁定, 这样再次调用 loadUserByUsername 返回的 JwtUser 在对锁定状态进行判断时会发现当前账户已经被锁定了

而当用户完成一次成功的登录后, 需要该计数从 redis 中清除

```java
// AuthenticationEventListener.java

@EventListener
public void onSuccess (AuthenticationSuccessEvent event) {
    JwtUser jwtUser = (JwtUser) event.getAuthentication().getPrincipal();
    String userKey =  constructUserKey(jwtUser.getUsername());
    redisUtil.delete(userKey);
}
```

上面提到了 IpUtil, 这是一个根据 request 获取 header 的工具类

```java
package icu.buzz.security.util;

import icu.buzz.security.constant.CommonConstant;
import icu.buzz.security.constant.HttpHeaderConstant;
import icu.buzz.security.exception.ServerInternalException;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Map;

@Component
public class IpUtil {
    private HttpServletRequest request;

    @Autowired
    public void setRequest(HttpServletRequest request) {
       this.request = request;
    }

    public String getIpAddr() {
        String ipAddr = null;
        boolean flag = false;
        for (String header : HttpHeaderConstant.IP_HEADERS) {
            ipAddr = request.getHeader(header);
            if (ipAddr != null && !ipAddr.isEmpty() && !ipAddr.equals(HttpHeaderConstant.UNKNOWN)) {
                flag = true;
                break;
            }
        }

        if (flag) {
            ipAddr = ipAddr.split(",")[0];
        } else {
            ipAddr = request.getRemoteAddr();
            if (ipAddr.equals(CommonConstant.LOCAL_IP)) {
                try {
                    ipAddr = InetAddress.getLocalHost().getHostAddress();
                } catch (UnknownHostException e) {
                    throw new ServerInternalException(Map.of("internal error", "cannot get ip"));
                }
            }
        }

        return ipAddr;
    }
}
```

获取真实 ip 确实不容易, 需要查询若干个头 ... 这些头已经被打包好封装到了 HttpHeaderConstant 中了

```java
package icu.buzz.security.constant;

public class HttpHeaderConstant {
    public static final String[] IP_HEADERS = {
            "X-Forwarded-For",
            "X-Real-IP",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_X_CLUSTER_CLIENT_IP",
            "HTTP_CLIENT_IP",
            "HTTP_FORWARDED_FOR",
            "HTTP_FORWARDED",
            "HTTP_VIA,REMOTE_ADDR"
    };
    public static final String UNKNOWN = "unknown";
}

```

>   这个是在网上抄的, 反正就这么多头, 挨个看看吧

现在的 JwtUser 可能被 locked 掉, 在 spring security 中体现为异常 LockException, 因此在 global exception handler 中也需要处理这个异常

```java
// GlobalExceptionHandler.java

@ExceptionHandler(LockedException.class)
public ResponseEntity<ErrorResponse> handleLockException(LockedException e, HttpServletRequest request) {
    return ResponseEntity
            .status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(ErrorCode.USER_LOCKED, request.getRequestURI(), Map.of("user locked", e.getLocalizedMessage())));
}
```

# Async

简单来说, 只要在方法上添加注解 @Async 即可将对应的方法体标记为异步方法, spring 在执行的时候会按照异步的方式执行 

>   其实还需要在配置类上添加 @EnableAsync 的注解

spring 官方给出的例子是比较简化的: [Getting Started | Creating Asynchronous Methods](https://spring.io/guides/gs/async-method) 并没有涉及异步任务相关的配置, spring 内部实现异步任务其实是将对应的任务分发给其他异步线程处理, 使得当前处理请求的线程可以提前返回

因此这里的配置也就是异步线程池的配置 + 异常处理

## 线程池

线程池的创建需要一系列参数, 第一次看的话应该会乱掉, 可以参考: [你管这破玩意叫线程池](https://mp.weixin.qq.com/s/b265zSAKYBtUESakVIWl1A) 先大概有点印象:

*   core pool size: 表示线程池中常驻线程的数量, 就算这几个线程是空闲的也不会被线程池回收掉 -> 这个参数会直接影响程序的 CPU 和内存的占用情况, 一般而言, 建议如果异步任务多为耗时短, 但频次高的任务, 该参数可以适当调整的大一点
*   maximum pool size: 表示线程池中线程数量的上限, 包括那些会被线程池动态创建/回收的线程 -> 一般而言这个参数和峰值下的任务负载相关, 如果设置的太大的话, 会产生线程资源的浪费, 但设置的太小也可能会导致异步任务的排队 (当然也可能导致异步任务被丢掉, 这个取决于拒绝策略)
*   task queue capacity: 当任务数目大于 core pool size 后, 任务会被保存在 task queue 中 -> 这个参数是用来应对业务突发的, 和业务的实际波动情况相关, 业务波动越剧烈, 越需要一个更大的 task queue

## exception handler

按照官网的说法: When an `@Async` method has a `Future`-typed return value, it is easy to manage an exception that was thrown during the method execution, as this exception is thrown when calling `get` on the `Future` result. With a `void` return type, however, the exception is uncaught and cannot be transmitted. You can provide an `AsyncUncaughtExceptionHandler` to handle such exceptions.

使用 @Async 注解的方法只有两种返回值类型, 要么是 Future, 要么是 void; 对于返回值是 Future 类型的, 当通过 Future.get 获取结果值时, 需要应用程序主动处理异常, 通过 try-catch 块可以捕获这种异常

然而对于返回值为 void 类型的异步方法, 大多数在调用结束后不会再对返回值进行检查, 应用程序无法直接处理异常, 因此这里还需要为异步任务配置一个 exception 处理通用异常

## one more thing

使用 @Async 有几个注意点:

*   不要在同一个类中调用 `@Async` 的方法, 不然方法会被正常顺序调用, 而不是异步方法: 这个其实和 spring 本身的代理模式机制相关, 如果在同一个类中调用异步方法, 那个对应的调用不会被 spring intercept 掉 -> 常规的使用方式就是在一个 bean 中调用另外一个 bean 中的异步方法
*   不要将 @Async 和 @Transaction 同时使用, @Async 会破坏 @Transaction 的原子性



