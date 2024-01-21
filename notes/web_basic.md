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



