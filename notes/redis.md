[redis 官网](https://redis.io/)

# 安装

## 源码

原来安装 redis 的时候都是通过下载源码，然后通过 cmake 进行编译

所有版本的 redis 都可以从 github 的 [release page](https://github.com/redis/redis) 上看到

首先通过 wget 获取最新版本的 tarball

>   在 2022/9/22 最新版本的 redis 为 7.0.5

```shell
$ wget https://github.com/redis/redis/archive/refs/tags/7.0.5.tar.gz
```

根据在 linux 下的安装习惯，建议将 redis 安装到 /user/local 下，解压

```shell
$ tar -xzvf 7.0.5.tar.gz
```

(这里需要 gcc 环境)

进入 redis 目录使用 make 编译得到可重定位目标文件

```shell
$ make
```

make install 获取可执行目标文件

```shell
$ make install
```

## ubuntu 已经自带了

源码编译有点费劲，反正 ubuntu 的官方库中已经包含了 redis，直接下就行

```shell
$ apt update
$ apt install redis
```

# redis-cli

正常情况下，需要和 redis-server 建立通信，需要使用 redis 指定的协议，建立基于 socket 的 TCP 连接

>   这应该就是其他语言和 redis 通信时使用的方式

为了方便调试，redis 官方提供了 [redis-cli](https://redis.io/docs/getting-started/#exploring-redis-with-the-cli)

# redis 的数据类型

目前看下来 redis 是基于键值对的，感觉就是一个 map

## keys

典型的，就是使用字符串作为键，不过 redis 也可以使用二进制序列表示一个键，官方说，甚至可以使用一个 JPEG 的内容作为一个键

redis 官方不建议使用太长的键，因为匹配的时间有点长，并且还有点占内存

redis 也不建议使用太短的键，不用为了那一点点的内存空间和匹配时间而过度压缩键名，一个好的键名就是清晰易懂的

官方建议的键名的格式为："object-type:id"，比如："user:10"(表示 uid 为 10 的用户)

再比如，如果对于某个域，可以写成："object-type:id.field"，比如："user:10.username"(表示 uid 为 10 的用户名)

## Strings

字符串类型应该是最常见的类型，特别的 set 和 get 命令就是针对字符串类型的 val 的操作

```shell
> set username buzz
> get username
buzz
```

从这里就能看出来了，操作 redis 和 操作 map 是类似的

此外，

通过对基本的 set 添加一些参数，可以实现特殊的功能，比如如果键已经存在了，就不添加了

```shell
# 参数 nx 表示只有在键不存在时才会添加
> set username buzz nx
# 参数 xx 表示只有在键存在时才会添加
> set username buzz xx
```

redis 使用 string 保存基本数据类型，特别的对于 integer，可以实现类似 inc 的操作

>   也许 redis 的 val 不存在基本数据类型，只有 string 类型

```shell
> set count 100
> incr count
(integer) 101
> incrby count 10
(integer) 111
```

注意到使用 incr 和 incrby 用来实现自增操作，redis 最强的地方是他保证了 incr 的原子性，多个线程同时调用了 incr，那么所有的这些 incr 操作都不会丢失，调用了几次，val 就会自增多少次

>   类似的还有 decr 和 decrby 操作

此外可以通过 mset 和 mget 同时分配、获取多个键

```shell
> mset a 1 b 2 c 3
OK
> mget a b c
1) "1"
2) "2"
3) "3"
```

# 一些命令

注意到下面所有的命令都可以在官方的[命令](https://redis.io/commands)中找到

## [exists](https://redis.io/commands/exists/)

该命令用来判断键是否存在，返回值只有两种 0 和 1，返回 0 就表示键不存在

```shell
> exists username
(integer) 1
```

注意这个命令的参数是可变的，即可以同时判断多个键是否存在，此时返回值就和查询的键的个数有关了

## [del](https://redis.io/commands/del/)

用来删除键，参数也是可变的，即可以同时删除多个键，返回值和当前命令删除掉的键的个数有关

如果当前删除的键不存在，那么返回的就是 0

```shell
> del username
(integer) 1
```

## [type](https://redis.io/commands/type/)

判断键的类型，因为键所存储的值的类型只有 `string`, `list`, `set`, `zset`, `hash` and `stream` 6 种

这个命令不支持多个参数，如果键不存在就返回 null

```shell
> del username
(integer) 1
> type username
none
```

## [set](https://redis.io/commands/set/)

set 指定的 key 为特定的 val，且 val 的类型为 string 类型，此时 redis 可以认为是一个 Map<String, String>

```shell
> set username buzz
OK
```

注意到，一个 key 只能映射到一个 val，所以如果 key 已经被赋值了，那么新的 set 将会覆盖原来的 val

通过对基本的 set 添加一些参数，可以实现特殊的功能，比如如果键已经存在了，就不添加了

```shell
# 参数 nx 表示只有在键不存在时才会添加
> set username buzz nx
# 参数 xx 表示只有在键存在时才会添加
> set username buzz xx
```

set 还可以通过其他参数限制键存在的时间

```shell
# 限制了键存在的时间为 5s
> set username buzz ex 5
# 限制了键存在的时间为 5ms
> set username buzz px 5
```

可以通过 [ttl](#ttl) 和 [pttl](#pttl) 查看键剩余可以存在的时间

## [expire](https://redis.io/commands/expire/)

设置一个键的过期时间，过期就意味着被删除

>   注意到，并不是所有类型的 val 对应的键都可以直接配置上一个过期时间

```shell
> expire username 10
(integer) 1
```

只有那些覆盖原来键本身的 val 的操作才会重置过期时间，比如 set 或 del，常见的比如 incr 或者 lpush 并不会重置过期时间

通过 [persist](#persist) 命令可以将一个键持久化，即重置过期时间；而通过 [rename](#rename) 命令对键进行重命名，新的键名将继承原有的过期时间

expire 命令也具有一些参数

```shell
> expire username 
```



## [persist](https://redis.io/commands/persist/)

将键的过期时间移除，如果键本身不存在过期时间，或者键压根就不存在，那么会返回 0

```shell
> persist username
(integer) 0
```

## [rename](https://redis.io/commands/rename/)

修改键名

如果新的键名已经被分配了，那么 rename 会将新键名原来的 val 进行覆盖，即将新键名原来的 val 删除掉，然后再进行 rename 操作

>   所以如果新键名的 val 很大，那么删除操作就可能不能在 O(1) 的时间内完成，此时 rename 操作的时间复杂度也就不能认为是O(1)的

```shell
> rename username name
OK
```

## [ttl](https://redis.io/commands/ttl/)

返回键的剩余存在时间单位为 s

因为键可能存在有效期，可能不存在，如果键本身不存在有效期，那么会返回 -1，如果键已经不存在了那么会返回 -2

```shell
> ttl username
(integer) -2
```

## [pttl](https://redis.io/commands/pttl/)

和 ttl 基本一致，最大的区别在于 pttl 返回的时间单位为毫秒

>   在 redis 中最小的单位就是毫秒
