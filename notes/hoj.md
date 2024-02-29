但愿能跑吧

按照官网的说法, 这还是个微服务, 整体上是前后端分离的框架, 后端是微服务的架构, 将评测服务单独抽取出来被封装为 hoj-judgeserver, 而原本的后端服务为 hoj-backend

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

