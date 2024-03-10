# HTTP

## request headers

一般涉及到定制化 request 的时候可以通过前端的 axios 或者某些 mock application, 当前在微服务的时代 spring 本身也是支持发起 request 的

### [Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)

Range 字段表示了 client 希望请求的资源的范围, 一般的格式为:

```sql
Range: bytes=[start]-[end]
```

表示请求对应字段的范围 start 到 end 内的字节

不过这里要注意的是, 使用 `Range` 获取部分 resource 的前提是 server 本身支持返回部分 resource, 如果 server 的 response 中的 [Accept-Ranges](#Accept-Ranges) 对应的 value 为 none, 此时表明 server 不支持仅返回部分 resource 

## response headers

所有的 response header 都是 server 在返回的时候需要添加的, 特别的在 spring boot 中操作 ResponseEntity 的方法 header 可以设置一个 header

```java
return ResponseEntity
                .status(HttpStatus.OK)
                .header(CONTENT_TYPE, "application/json")
    			.body(...);
```

### [Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type)

Content-Type 对应了实际返回的内容的数据类型, 一般这个字段对应了一个 [MIME type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types)

### [Content-Length](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Length)

Content-Length 的 value 为 response body 的大小, 单位为字节

### [Accept-Ranges](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Ranges)

这个 header 表明了 server 是否支持仅返回 request 中的部分 resource, 这个 header 通常可以取到两个值:

*   `bytes`: 表示 server 支持仅返回部分 resource, client can request specific byte ranges of the resource
*   `none`: 表示 server 不支持仅返回部分 resource

当取值为 `bytes` 时, client 的下一个 request header 中可以包括名为 [Range](#Range) 的 key, 以获取 specific byte ranges of the resource

### [Content-Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Range)

当 client 的 request 中包含了 [Range](#Range) 字段, 请求了部分 resource 的时候, server 获取对应部分的内容, 并在 response header 中添加 `Content-Range` 表明当前返回的文件范围, 一般格式为:

```sql
Content-Range: bytes [start]-[end]/[total]
```

其中 start-end 对应了返回的文件的范围, 而 total 表示了整个文件的大小

当使用 Content-Range 时, HTTP status code 经常被设置为 206 -> Partial Content, 表示仅仅返回了部分 resource

# CSRF

不太好说明类型, spring 官网是用一个例子说明的: [Cross Site Request Forgery (CSRF) :: Spring Security](https://docs.spring.io/spring-security/reference/features/exploits/csrf.html)

CSRF 的问题出现在 cookie 中, 攻击者并不需要知道 cookie 的内容, 当 client 端没有通过 logout 主动释放关闭一个 session, cookie 还保存在 client 端, 当 client 访问到恶意网站后, js 可以主动发起一个到原网站的请求, 此时 browser 会携带由 cookie 信息进行身份验证

在使用 jwt 标识身份信息的场景中, 其实不太需要关心这个问题 -> 只要 jwt 不保存在 cookie 中 (in-memory 或 local storage 中), 此时在 spring security 的配置中, 直接把这个配置 disable 掉

# RSA key pair

在某些场景中需要用到非对称加密, 比如在最新的 jjwt 中, 要求 jwt 需要使用 private-key 签名, 而使用 public-key 进行校验

一般而言可以借助 ssh-keygen 生成这个 key, 而在一般的应用中, 很多时候最好直接生成 .pem 格式的 key 文件, 这个时候需要用到 openssl 这个工具

```shell
# 在当前目录下生成一个私钥
$ openssl genrsa -out ./private-key.pem 2048 
```

对应的公钥通过私钥生成

```shell
$ openssl rsa -in private-key.pem -pubout -out public-key.pem
```

>   如果为了私钥的安全甚至可以生成一个加密了的私钥 ... 混合加密 ?



