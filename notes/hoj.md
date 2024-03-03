但愿能跑吧

按照官网的说法, 这还是个微服务, 整体上是前后端分离的框架, 后端是微服务的架构, 将评测服务单独抽取出来被封装为 hoj-judgeserver, 而原本的后端服务为 hoj-backend



# log

*   2.29: 从登录开始看: PassportController.login() -> PassportServiceImpl.login() -> PasswordManager.login():

    1.   进行参数列表的校验 (抛异常)
    2.   校验当前 ip 下, 当前用户的登录次数 (抛异常)
    3.   根据 username 查询用户 (抛异常)
    4.   对比 password -> md5 hash (抛异常)
    5.   获取 user status (用户进入黑名单, 抛异常)
    6.   由 uuid 生成 jwt, 添加两个 header, Authorization 和 Access-Control-Expose-Headers
    7.   
    
    
    
     

# backend

给出了三个配置, application.yml, application-dev.yml, application-prod.yml

涉及到微服务的结构, 还给出了配置文件 boostrap.yml, 看下来主要配置的是 nacos, 项目名, 使用的 profile

>   具体的 bootstrap.yml 的作用可以查看 [Spring Cloud Context: Application Context Services :: Spring Cloud Commons](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/application-context-services.html) 和 [Spring Configuration Bootstrap vs Application Properties | Baeldung](https://www.baeldung.com/spring-cloud-bootstrap-properties)
>
>   简单看下来, application.yml 是用来配置应用程序的上下文的, 而 bootstrap.yml 是: **loading configuration properties from the external sources**
>
>   看他们给出的示例中, 一般 bootstrap 也就是指定一下项目名, 使能的 profile, 然后就是 spring cloud 的相关配置了

默认 backend 使能的是 prod, 项目 down 下来修改为 dev, 才能正常使用

## RedisTemplate

在 spring 中一般不会直接使用 jedis 操作 redis 数据库, 而是使用 RedisTemplate, [Working with Objects through RedisTemplate :: Spring Data Redis](https://docs.spring.io/spring-data/redis/reference/redis/template.html)

在 spring 中想要使用的各个对象, 都是通过 Bean 管理的, RedisTemplate 一般通过通过 RedisConfig 类完成注入

```java
// config.RedisConfig.java

package icu.buzz.redis.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean("String2StringTemplate")
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // keys and value are all string
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());

        // redis can hold hash table as value: key -> hash table
        // this configuration is used to configure hash table key and value serializer
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

这里默认给出了一个 key 和 value 均为 String 类型的 RedisTemplate, 默认 RedisTemplate 使用 jdk 自带的 serializer 序列化 redis 的 key 和 value; 因此保存在 redis 中之后, 可能出现乱码 ...

因为 key 和 value 都是 string, 因此这里主动配置了 string serializer

```java
// controller.RedisController.java

package icu.buzz.redis.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/redis")
public class RedisController {

    @Autowired
    @Qualifier("String2StringTemplate")
    private RedisTemplate<String, String> template;

    @PostMapping("/set/{key}/{value}")
    public String set(@PathVariable String key, @PathVariable String value) {
        template.opsForValue().set(key, value);
        return "success";
    }

    @GetMapping("/get/{key}")
    public String get(@PathVariable String key) {
        return template.opsForValue().get(key);
    }
}
```

>   拿 ApiPost 试试吧, 后面就能在 Redis Insight 中看到 key 和 value 了

