尝试在一个月之内突击完 netty, 大概率是做不了什么项目的...

# NIO

被称为 none-blocking io, NIO 由三部分组成:

*   Buffer: NIO 中用来保存数据的单位, 执行读取时会将数据从 channel 中读入到 buffer 中; 执行写操作时会将 buffer 中的数据写入到 channel 中
*   Channel: 在 NIO 中, 各种文件, socket 都被抽象为一个 Channel, 可以类比 IO stream, 区别是 Channel 是支持同时读写的, 再也不需要纠结于 InputStream 还是 OutputStream 了
*   Selector: NIO 不阻塞的核心, 一个 Selector 上可以注册多个 Channel, 注册时还应该明确需要关注的事件 -> 后面具体说

## Buffer

NIO 中的 Buffer 具有如下继承关系

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/01/27/20:43:57:buffer.png)

从命名上可以看出来, 是按照不同的数据类型区分的, 更为具体的, 每种数据类型也具有很多实现类, 这里不过多赘述, 在使用的时候可以简单的认为 buffer 就是对应类型的数组

在 buffer 中具有四种很重要的成员变量:

```java
// Buffer.java

// Invariants: mark <= position <= limit <= capacity
private int mark = -1;
private int position = 0;
private int limit;
private int capacity;
```

*   capacity 表明 buffer 的大小, 在声明一个 buffer 时确定大小

*   limit 表明数据边界, 在 buffer 处于写模式下, limit 会限制能写入的数据的量, 一般而言会将 limit 设置和 capacity 相同; 在 buffer 处于读模式下, limit 会限制可以读的数据的量, 也就是之前写入数据的量

    >   一个新创建的 buffer 默认是写模式, 在经过一系列写入操作后, 通过调用方法 filp 转变为读模式, 此后在读模式下, 可读的数据量不会超过之前写入的数据量, 即不会读到 junk bytes 

*   position 表明下一个被操作的位置, 即下一个可以被写/被读的位置

    >   在写模式下, 调用方法 flip 会让 limit 取 position, 再让 position 清零

*   mark: buffer 支持将 position 的重置为 mark

一般而言, 不会通过构造方法创建一个 Buffer (ByteBuffer 本身都是 abstract 的), 而是通过调用静态方法完成创建

```java
// ByteBuffer.java

/**
 * Allocates a new direct byte buffer.
 *
 * <p> The new buffer's position will be zero, its limit will be its
 * capacity, its mark will be undefined, each of its elements will be
 * initialized to zero, and its byte order will be
 *
 * @param  capacity The new buffer's capacity, in bytes
 * @return  The new byte buffer
 * @throws  IllegalArgumentException
 *          If the {@code capacity} is a negative integer
 */
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}

/**
 * Allocates a new byte buffer.
 *
 * <p> The new buffer's position will be zero, its limit will be its
 * capacity, its mark will be undefined, each of its elements will be
 * initialized to zero, and its byte order will be

 * @param  capacity The new buffer's capacity, in bytes
 * @return  The new byte buffer
 * @throws  IllegalArgumentException
 *          If the {@code capacity} is a negative integer
 */
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw createCapacityException(capacity);
    return new HeapByteBuffer(capacity, capacity, null);
}
```

方法 allocate 会在堆区进行 Buffer 的分配, 而 allocateDirect 会直接在物理内存中进行 Buffer 的分配

一般而言通过方法 put/get 表示填充/获取数据; 调用方法 flip() 会让 buffer 从写模式切换到读模式; 而调用方法 clear() 会清空 buffer, 并**隐式地将 buffer 从读模式切换到写模式**

>   调用 clear 后, position 被置为 0, limit 被置为 capacity

```java
public static void main(String[] args) {
    // integer buffer with capcaity 5
    IntBuffer buffer = IntBuffer.allocate(5);
    for (int i = 0; i < buffer.capacity(); i++) buffer.put(i);
    // mode switch
    buffer.flip();
    // print 0 ~ 4
    while (buffer.hasRemaining()) System.out.println(buffer.get());
}
```

一个 Buffer 可以通过调用方法 asReadOnlyBuffer() 获取一个只读的 buffer

## Channel

在各种 Channel 中, 有三种最常用的 Channel:

*   FileChannel: 文件操作相关
*   SocketChannel, ServerSocketChannel: TCP 相关, 类比 Socket 和 ServerSocket
*   DatagramChannel: UDP 相关

channel 需要和 buffer 结合使用, 通过调用 read 方法实现向 buffer 写入, 或调用 write 将 buffer 中的内容写入 channel

之前需要使用 IO stream 完成的操作均可以使用 Channel 代替, 为了说明 channel 是全双工的, 这里针对同一个文件开启channel, 首先读取文件内容, 随后利用相同的 channel 进行写入

```java
private static void read_and_write(String words) {
    Set<OpenOption> options = Set.of(StandardOpenOption.CREATE, StandardOpenOption.READ, StandardOpenOption.WRITE);
    try (FileChannel channel = FileChannel.open(Paths.get("./test.txt"), options)) {
        ByteBuffer buffer = ByteBuffer.allocate(1);
        int len = channel.read(buffer);
        byte[] bytes = new byte[1024];
        if (len != -1) {
            while (len != -1) {
                buffer.flip();
                buffer.get(bytes, 0, len);
                String original = new String(bytes, 0, len, StandardCharsets.UTF_8);
                System.out.print(original);
                buffer.clear();
                len = channel.read(buffer);
            }
        } else System.out.println("no existing works");
        bytes = words.getBytes(StandardCharsets.UTF_8);
        // switch back to write mode
        buffer.clear();	
        int idx = 0;
        while (idx < bytes.length) {
            int size = buffer.remaining();
            buffer.put(bytes, idx, size);
            buffer.flip();
            int write = channel.write(buffer);
            if (write != size) System.out.println("short count");
            buffer.clear();
            idx += size;
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

本例中使用静态方法 FileChannel.open 获取一个 Channel, 可以看到, 在申请这个 channel 的时候, 使用的标志位是 CREATE/READ/WRITE 这一点有点类似在其他语言中获取 file descriptor 的过程

随后创建了一个 ByteBuffer 用来和 FileChannel 交互, 这里的 ByteBuffer 大小为 1, 即只能保存一个字节, 实际中不应该这么小, 在读写大文件时应该创建更大的 ByteBuffer, 这里使用小 buffer 主要目的是为了说明, 在 buffer 一次性不能读完文本中的所有字节时, 需要循环读取

要注意的是, 只有在 ByteBuffer 为写模式下, channel 才可以调用 read 方法进行 buffer 的填充, 而为了打印读取到的内容, 需要将 buffer 转化为读模式, 即调用 flip() 方法, 而为了下一轮此的读操作, 需要重新将 buffer 恢复为写模式, 即调用 clear() 方法

>   实际中 buffer 并没有所有的写模式和读模式, 在看了 buffer 的成员变量之后和各个方法之后可以看到, buffer 就是通过 position, limit, capacity 维护保存的缓存, 而通过调用方法 flip 和 clear 修改这些成员变量
>
>   这里仅仅是为了方便理解, 才对 buffer 进行写/读模式的区分

在完成打印后, 再利用相同的 ByteBuffer 将入参 @param: words 写入 FileChannel, 同样的, 因为 buffer 大小有限, 因此在写入文件的时候也是需要通过循环写入的, 基本原理还是相同, 需要 buffer 不断在写/读模式下进行切换

NIO 提供了很强大的工具, 支持 scatter 和 gather, channel 可以将内容填写到多个 buffer 中, 也可以从多个 buffer 获取数据写入 channel; 并且还提供了十分简单的 API, 还是原来的 read/write 方法, channel 提供了重载方法, 通过将多个 buffer 添加到一个数组中, 可以实现一次性按照顺序写入多个 buffer, 或者按照顺序读取多个 buffer

## Selector

一般而言, 可以认为一个 selector 对应了一个线程, 将 Channel 和对应的事件注册到 selector 后, 可以通过访问 selector 判断事件是否发生, 从而实现一个 selector 管理多个 channel

在典型的网络场景中, selector 可能同时监听 socket 连接事件, 读/写事件, 在某次轮询中, 可以获取已经发生的事件 -> SelectionKey, 并获取事件对应的 channel

一般而言 selector 关注四种事件类型:

*   SelectionKey.OP_ACCEPT: 接收连接事件, 一般 ServerSocketChannel 在注册到 selector 时使用
*   SelectionKey.OP_CONNECT: 完成连接事件, 一般 SocketChannel 在注册到 selector 时使用
*   SelectionKey.OP_READ: 可读事件, 表示可以从对应的 Channel 中读取数据
*   SelectionKey.OP_WRITE: 可写事件, 表示可以将数据写入到对应的 Channel

Selector 一般也不是通过构造方法创建的, 更多的是调用静态方法 open() 获取一个 Selector

*   select() => 获取对应事件发生的 channel 的个数, 这个方法是阻塞的, 即至少一个 channel 发生了对应的事件才会返回
*   select(long timout) => 相当于设置了一个超时, 在指定超时事件后必然返回
*   selectNow() => 非阻塞的, 可能返回 0, 即没有任何一个 channel 的事件发生
*   selectedKeys() => 直接获取一个所有事件已经发生 keys => 可以间接获取对应的 channel

一般而言通过一个 selector 在某一次轮询在中有如下结构:

```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key != null) {
        if (key.isAcceptable()) {
            // ServerSocketChannel 接收了一个新连接
        } else if (key.isConnectable()) {
            // 表示一个新连接建立
        } else if (key.isReadable()) {
            // Channel 有准备好的数据，可以读取
        } else if (key.isWritable()) {
            // Channel 有空闲的 Buffer，可以写入数据
        }
    }
    keyIterator.remove();
}
```

## TCP with NIO

这里需要实现一个带有转发功能的多用户的聊天程序, 从某种程度上, NIO 实现一个支持多 client 的聊天程序其实比 BIO 更简单一点 ...

为了尽可能简化, 这里并没有为每个 IO 分配一个 handler 线程处理, 转发功能在 server 线程本身实现; 而为了保证 client 可以持续读入输入, 这里每个 client 开辟了两个线程, 一个用来读取用户输入, 并发向 server; 另一个线程被阻塞, 用来持续获取来自 server 的输入

### server

默认情况下, server 在开启时仅维护了一个 ServerSocketChannel, 同时使用 selector 管理

```java
public ServerChannelOperation(int port) {
    try {
        this.server = ServerSocketChannel.open();
        // non-blocking server channel
        server.configureBlocking(false);
        server.socket().bind(new InetSocketAddress(port));
        this.selector = Selector.open();
        // ServerSocketChannel needs OP_ACCEPT
        server.register(selector, SelectionKey.OP_ACCEPT);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

在完成初始化操作后, server 开始持续执行监听, 等待 client 的连接请求或 IO 请求

```java
public void listen() {
    try {
        while (true) {
            // block here
            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                if (key.isValid()) {
                    if (key.isAcceptable()) {
                        SocketChannel client = server.accept();
                        client.configureBlocking(false);
                        client.register(this.selector, SelectionKey.OP_READ);
                    } else if (key.isReadable()) handle(key);
                }
                iterator.remove();
            }
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

如果当前是一个 ACCEPT 事件, 则调用 accept() 获取 SocketChannel, 这里需要显示的将 client 设置为 none-blocking, 并交给 selector 托管, 标记的事件类型为 READ

如果当前是一个 READ 事件, 表示当前存在一个 client 向 server 发送了数据, 此时 server 调用 handle 进行日志记录并转发给其他 client

这里的 listen 整体上使用的就是上面的框架, 通过迭代器处理各个 SelectionKey, 每次处理完一个事件将该事件从集合中删除

```java
private void handle(SelectionKey clientKey) {
    SocketChannel client = (SocketChannel) clientKey.channel();
    try  {
        ByteBuffer buffer = ByteBuffer.allocate(8);
        int len = client.read(buffer);
        if (len > 0) {
            System.out.print("from " + client.getRemoteAddress() + ":");
            while (len > 0) {
                buffer.flip();
                byte[] bytes = new byte[buffer.remaining()];
                buffer.get(bytes);
                forward(bytes, client);
                System.out.print(new String(bytes, StandardCharsets.UTF_8));
                buffer.clear();
                len = client.read(buffer);
            }
            System.out.println();
        }
    } catch (IOException e) {
        try {
            System.out.println(client.getRemoteAddress() + " off line");
            clientKey.cancel();
            client.close();
        } catch (IOException ex) {
            throw new RuntimeException(ex);
        }
    }
}
```

这里故意设置一个很小的 ByteBuffer, 通过一个 while 循环保证 server 可以一次性完成 client 输入的读入, 每个读入的输入都会依次调用转发方法 forward 完成转发

```java
private void forward(byte[] bytes, SocketChannel client) {
    Set<SelectionKey> keys = selector.keys();
    for (SelectionKey key : keys) {
        if (key.channel() instanceof SocketChannel socketChannel && socketChannel != client) {
            ByteBuffer tmp = ByteBuffer.wrap(bytes);
            try {
                socketChannel.write(tmp);
            } catch (IOException e) {
                try {
                    System.out.println(socketChannel.getRemoteAddress() + " off line");
                    key.cancel();
                    socketChannel.close();
                } catch (IOException ex) {
                    throw new RuntimeException(ex);
                }
            }
        }
    }
}
```

方法 forward 会向除了输入 client 之外的所有其他 client 进行转发, 这是群聊功能的关键部分, 至于 client 就相对简单了很多 (除了两个线程之外)

```java
public ClientChannelOperation(int port) {
    try {
        this.client = SocketChannel.open();
        this.client.configureBlocking(false);
        this.selector = Selector.open();
        this.client.register(selector, SelectionKey.OP_CONNECT);
        this.client.connect(new InetSocketAddress("127.0.0.1", port));
        this.waitForConnect();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

尽管每个 client 都只需要管理一个 socket channel, 但还是使用了 selector 管理, selector 首先监听 CONNECT 事件, 完成 client 到 server 的连接建立, 随后调用 waitForConnect() 等待连接完成

```java
private void waitForConnect() throws IOException {
    selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> iterator = keys.iterator();
    while (iterator.hasNext()) {
        SelectionKey key = iterator.next();
        if (key.isValid() && client.finishConnect()) {
            client.register(selector, SelectionKey.OP_READ);
            Thread readThread = new Thread(this::read);
            readThread.start();
        }
        iterator.remove();
    }
}
```

在连接完成之后重新为 client 配置监听事件, selector 随后需要监听 READ 事件, 等待来自 server 的转发, read 方法也是一个阻塞的方法, 和 server 类似

```java
private void read() {
    try {
        while (true) {
            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                if (key.isValid() && key.isReadable()) {
                    ByteBuffer buffer = ByteBuffer.allocate(8);
                    int len = client.read(buffer);
                    if (len > 0) {
                        System.out.print("from " + client.getRemoteAddress() + ":");
                        while (len > 0) {
                            byte[] bytes = new byte[buffer.remaining()];
                            buffer.flip();
                            buffer.get(bytes, 0, len);
                            String msg = new String(bytes, 0, len, StandardCharsets.UTF_8);
                            System.out.print(msg);
                            buffer.clear();
                            len = client.read(buffer);
                        }
                        System.out.println();
                    }
                }
                iterator.remove();
            }
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

类似的, 这里的 buffer 也设置了 8 个字节, 并通过 while 循环持续从 server 获取输入, 最后的话就是 write 方法了, 用来向 server 发送数据

```java
public void write(String msg) throws IOException {
    ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes(StandardCharsets.UTF_8));
    client.write(buffer);
}
```

>   200 行以内实现群聊

# netty

前面的 NIO 通过原生的 channel, selector, buffer 机制实现了一个单线程的 server, 相比 BIO, 还是复杂很多的

netty 基于 NIO 进一步进行了封装, 实现网络连接管理和业务逻辑的解耦, 本着最好的学习资料来自于官方的原则, 入门阶段就当成是对对官方资料进行翻译吧

## discard server

[RFC 863 - Discard Protocol](https://datatracker.ietf.org/doc/html/rfc863) 应该说是最简单的协议了, client 只管发就好了, server 不管收到了什么消息都直接拒绝掉

### DiscardServerHandler

netty 将所有的业务逻辑都使用 xxxHandler 抽象, 对于本例需要实现的是 DiscardServerHandler, 在这个 handler 中 server 需要丢弃所有的消息

```java
// DiscardServerHandler.java

package icu.buzz.example;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.ReferenceCountUtil;

// it is the handler's responsibility to release any reference-counted object passed to the handler
// in this case -> msg
public class DiscardServerHandler extends ChannelInboundHandlerAdapter {

    // this method will be called after server receive a message from client
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf)msg;
        try {
            while (in.isReadable()) {
                System.out.print((char)in.readByte());
                System.out.flush();
            }
        } finally {
            // release the buffer
            ReferenceCountUtil.release(msg);
            // or in.release();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        // close channel after exception
        ctx.close();
    }
}
```

实现类 DiscardServerHandler 继承了 ChannelInboundHandlerAdapter, 其是 ChannelInboundHandler 的一个实现类, 提供了基本的处理逻辑

>   除了 Inbound 当然还有 Outbound, 后面再说吧
>
>   基本上大部分的 Handler 都可以通过继承 xxxAdapter 实现

实现 discard 的关键在于重写方法 channelRead, 当 client 向 server 发送新的 message 后, netty 会调用该方法并将 message 作为一个参数传入

>   默认情况下, msg 是一个 ByteBuf, 实际的业务处理逻辑中, msg 可以为各种 POJO
>
>   在本例中参数 ctx 没什么用, 毕竟直接就将消息忽略了... 但这个 ctx 其实还是很有用的

ByteBuf 是一个 reference-counted object, 在使用完 msg 后需要主动调用方法 relese, 本例中使用的是工具类 ReferenceCountUtil 提供的静态方法 release 完成对象的释放

官方的话是: it is the handler's responsibility to release any reference-counted object passed to the handler. 一般而言 channelRead 需要按照如下方式实现:

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    try {
        // Do something with msg
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

除了 channelRead 之外, 本例还重写了方法 exceptionCaught(), 从名字中就能看出来, 这个方法是用来异常处理的; 在进行异常处理时, 需要完成异常的记录, 并在该方法内关闭出现异常的 channel, 更特殊的还可以在关闭连接之前向 client 端返回错误信息 (HTTP server);

### DiscardServer

处理业务处理逻辑之外, netty 还需要对连接进行配置

```java
package icu.buzz.example;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class Server {
    private final int port;

    public Server(int port) {
        this.port = port;
    }

    public void run() throws InterruptedException {
        // EventLoopGroup => multi selectors
        // boss handles incoming connections
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        // workers handle traffic of connections (boss will register the accept connections to workers)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            // helper class -> configure ServerSocketChannel
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class) // NioServerSocketChannel is a ServerSocketChannel with netty feature
                    // specify a handler for each SocketChannel, ChannelInitializer is used to configure SocketChannel
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            // use pipeline to describe operations for SocketChannel
                            socketChannel.pipeline().addLast(new DiscardServerHandler());
                        }
                    })
                    // this parameter has the same meaning as @param backlog in ServerSocket (maximum queue length for incoming connection indications)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    // .option is used to configure NioServerSocketChannel, .childOption is used to configure channel accepted by ServerChannel (NioSocketChannel)
                    // this parameter is used to send periodically packet to detect is client still alive
                    // the interval of ping-pang packet is specified by os itself (maybe not a good idea for a project)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // bind ServerSocket to the port, and fire up for incoming connections
            ChannelFuture future = bootstrap.bind(this.port).sync();

            // the server will block here until the ServerSocket is closed
            // graceful shutdown !
            future.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

}
```

>   这部分看起来代码量更多, 但它其实是通用的

除开构造方法不看, 这里只关心 run 方法

首先构造了两个 NioEventLoopGroup 对象: bossGroup 和 workerGroup; 在有关 netty 的介绍中基本上都是使用 boss 和 worker 进行命名的, 但实际 new 对象时候可以发现其实创建的是同一种对象; 根据官网的说法, NioEventLoopGroup 是一个 multithreaded event loop that handles I/O operation, 可以认为每个 group 都包含了多个 thread (EventLoop); 从习惯上, 使用 bossGroup 用来处理来自 client 的连接请求, 而使用 workerGroup 处理每个 connection 的 IO 请求

>   顶层的父类是 EventLoopGroup, 一般而言可以通过构造方法配置每个 Group 中包含的线程数和 channel 与线程的对应关系

随后创建了一个 ServerBootstrap 对象, 这个对象用来对 server 进行配置, 并启动 server; 

每个 ServerBootstrap 中都有一个 parentGroup 和一个 childGroup, 通过调用 group 方法完成 bossGroup 和 workerGroup 的绑定, 此后 bossGroup 被配置为 parentGroup, 而 workerGroup 被配置为 childGroup, 在后续的配置中, 调用方法 childxxx 进行对 childGroup 的配置

方法 channel() 配置用来处理来自客户端连接请求的 channel 类型, 这里配置为 NioServerSocketChannel, 这里可以粗略的认为其对应了在原生 NIO 中的 ServerSocketChannel

方法 childHandler() 故名思意, 就是配置实际的业务处理逻辑; 要注意的是在 NIO 中 channel 是 IO 操作的对象, 这里将 SocketChannel 作为泛型参数进行配置, 创建的 ChannelInitializer 对象就是一个针对 SocketChannel 的封装, 封装后的 "channel" 会被 workerGroup 管理

>   要特别注意的是, 这里使用的 SocketChannel 是 netty 中的一个接口, 而不是原生 NIO 中的 SocketChannel 类

实际的处理逻辑通过调用 pipeline() 的方式进行配置, 从名字中也能看出来, 每个 handler 之间是存在先后顺序的

在此基础上, 还可以通过 option() 和 childOption() 分别对 bossGroup 和 workerGroup 进行配置; 比如上面的 ChannelOption.SO_BACKLOG 对应了在 ServerSocket 中的参数 @param: backlog, 在多个 client 连接建立请求同时到达时, server 需要按序完成连接的建立, 在连接建立的过程中, 新的请求到达后, 会被放入队列中缓存, 这里的 backlog 配置的就是队列大小 

ChannelOption.SO_KEEPALIVE 表明 server 会定期向 client 发送心跳包, 确认连接的建立; 基本上平时会用得上的参数都在 ChannelOption 中定义了

注意到启动一个 server 只需要绑定端口号, 方法 sync 用来确保在方法返回后, 端口一定完成了绑定

>   方法 bind 是一个异步的方法, 返回值是一个 futrue, 会在实际的端口完成绑定之前就返回, sync() 会强制线程阻塞等待绑定完成

最后调用 closeFuture() -> graceful shutdown

## echo server

这个应该是除了 discard server 之外, 最简单的了 [RFC 862 - Echo Protocol](https://datatracker.ietf.org/doc/html/rfc862)

为了实现 echo server 只需要修改 handler 即可, 在上面的 DiscardServerHandler 中, channelRead() 方法只会不断打印来自 client 的消息, 为了实现一个 echo server 只需要在 channelRead() 中额外添加将消息返回给 client 的逻辑即可

```java
// EchoServerHandler.java

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // message will be released automatically after write
    ctx.write(msg);
    // message will be buffered until flush is called
    ctx.flush();
    // or ctx.writeAndFlush(msg);
}
```

ChannelHandlerContext 可以通过调用 write 向 client 发送消息, 由于可能存在缓存的原因, 因此只有在调用 flush 之后才会将消息发送给 client

注意到在 channelRead 中没有调用 release 方法进行 message 的释放, 因为在调用 write 之后 netty 就会隐式的完成对象引用的释放

## time server

这是官方给出的最后一个示例, 后续的各种改进都是在 timer server 上改进, 这里实现的是 [RFC 868 - Time Protocol](https://datatracker.ietf.org/doc/html/rfc868)

类似的需要修改的也只有 handler

### TimeServerHandler

一个 timer server 只需要向 client 返回时间消息, 因此这里不需要维护 client 和 server 的连接关系, 因此这里不需要重写 channelRead 方法, 而需要重写 channelActive, 这个方法会在连接完成建立后调用

```java
// TimeServerHandler.java

@Override
public void channelActive(final ChannelHandlerContext ctx) {
    final ByteBuf time = ctx.alloc().buffer(4);
    time.writeInt((int) (System.currentTimeMillis() / 1000L));

    final ChannelFuture f = ctx.writeAndFlush(time);
    f.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) {
            assert f == future;
            ctx.close();
        }
    });
}
```

在 NIO 中, 操作对象是 channel, 而用来在 channel 中进行数据传输的一定是 buffer; 在 netty 中使用的是 ByteBuf 对象, 这是 netty 提供的一个改进的 ByteBuffer

通过 ChannelHandlerContext 的 alloc 方法进行获取一个 ByteBufAllocator, 通过调用 buffer 方法可以返回一个 ByteBuf 对象

ByteBuf 最大的优点在于, 其在使用的时候不需要再调用 flip 方法了, ByteBuf 使用了两个 pointer 维护读写关系, 一个 read pointer 维护当前已经读取到的位置, 一个 write pointer 维护当前已经写入的位置

netty 的一个特点是异步非阻塞的 IO, 因此 write 操作不会立刻完成, 其返回值是一个 ChannelFutrue 对象; 因为 timer server 再返回一次时间消息后, 就不需要继续维护和 client 的连接关系了, 因此 timer server 需要在写入完成之后及时关闭 channel 连接; 而由于非阻塞的特性导致函数 channelActive 返回后, 可能还没有完成写入

类似其他语言中对于异步处理的操作, 这里 netty 为 ChannelFuture 添加了一个回调函数, 在 java 中自然需要一个对象进行封装, 这里的对象就是 ChannelFutureListener, 在行为结束后, 会调用 operationComplete 方法, 这里也是释放 channel 的最佳地点

>   要注意的是, 行为结束后释放资源是一个很常见的操作, 因此 netty 在 ChannelFutureListener 中通过成员变量的方式提供了一个 listener, 因此上面的操作等价于:
>
>   ```java
>   future.addListener(ChannelFutureListener.CLOSE);
>   ```
>
>   成员变量 CLOSE 就是一个 ChannelFutureListener 的一个实现类

### time client

由于上面的 handler 只会返回 4 个字节, 表示时间, 因此这里再使用 telnet 连上去就看不懂了 -> 文本不可读啊; 为了检查程序的正确性, 这里还需要提供一个 time client 用于正确解析这 4 个字节

netty 提供的是一套完整的异步 NIO 解决方案, 因此在 client 端也不需要使用原生的 NIO 进行操作, 可以使用类似 server 端的配置对 client 进行配置

```java
// TimeClient.java

package icu.buzz.example;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;


public class TimeClient {
    public static void main(String[] args) throws InterruptedException {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        // client only need one thread group
        EventLoopGroup worker = new NioEventLoopGroup();
        try {
            // client need to be configured by Bootstrap (server is configured by ServerBootstrap)
            // Bootstrap is used for non-server channel => client side mostly
            Bootstrap bootstrap = new Bootstrap();
            // bootstrap with only one EventLoopGroup => this group will be used as both boss and worker
            bootstrap.group(worker)
                    // NioSocketChannel is used as SocketChannel at client side
                    .channel(NioSocketChannel.class)
                    // client-side SocketChannel does not have a parent
                    .option(ChannelOption.SO_KEEPALIVE, true);
            // configure the handler for SocketChannel
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    // use pipeline to describe operations for SocketChannel
                    // add a TimeDecoder to deal with fragmentation issue
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });

            // use connect() on client side (bind on server side)
            ChannelFuture future = bootstrap.connect(host, port).sync();
            // wait until the connection is closed
            future.channel().closeFuture().sync();
        } finally {
            worker.shutdownGracefully();
        }
    }
}
```

还是一样的, 使用 NioEventLoopGroup 配置一个循环事件组 (反正一个 client 也会用这种方式连接一个 server 就是了)

注意到在 client 端使用的 Bootstrap 作为配置类, 而不是 ServerBootstrap, 还是同样的 group 方法, client 不需要将连接建立和 IO 请求分为两部分了, 因此 group 方法只有一个参数, 表明这个 group 既是 bossGroup 也是 workerGroup

这里配置的 channel 是 NioSocketChannel, 这里可以类比在原生 NIO 中的 SocketChannel; 参数 option 和 server 类似, 都会定期向 server 发送心跳包确保连接的稳定性 (但反正建立一次连接并返回时间后, 连接就会被 server 主动关闭就是了)

在 client 端, 连接管理和业务逻辑也是解耦的, 这里也是通过调用 handler 方法进行业务逻辑的配置, 主要的逻辑都放在了 TimeClientHandler 中

在完成配置后, 通过调用 connect 方法发起向 server 的连接 (对比在 server 端通过 bind 绑定端口号)

剩下的关键就在于 TimeClientHandler 的实现了

### TimeClientHandler

类似 server handler 的写法, 这里的 TimeClientHandler 也是继承了 ChannelInboundHandlerAdapter

```java
// TimeClientHandler.java

package icu.buzz.example;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.ReferenceCountUtil;

import java.util.Date;

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    // read time and close the connection
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf m = (ByteBuf) msg;
        try {
            long currentTimeMillis = m.readUnsignedInt() * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        } finally {
            m.release();
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

channel 底层进行数据传输的时候都是字节, 因此这里的 msg 也是一个 ByteBuf, 这个类提供了一个方法可以直接读取 4 个字节, 直接调用 readUnsignedInt() 即可完成时间的构造

最后不要忘了释放资源 -> m.release 或者 ReferenceCountUtil.release

### fix bug

上述代码在大部分情况下都没有问题, 但有的时候还是会抛异常, 说到底 TCP/IP 是一个 stream-based transport protocol, 数据会以字节的形式保存下来, 在缓存队列中不会对不同应用的数据包进行区分, 因此程序在读取数据的时候还需要进行数据分段, os 并不会保证数据分段的正确性

#### first solution

最简单的方式就是提升 ByteBuf 的作用域

```java
// TimeClientHandler.java

public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private ByteBuf buf;
    
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        buf = ctx.alloc().buffer(4);
    }
    
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) {
        buf.release();
        buf = null;
    }
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf m = (ByteBuf) msg;
        buf.writeBytes(m);
        m.release();
        
        if (buf.readableBytes() >= 4) {
            long currentTimeMillis = buf.readUnsignedInt() * 1000L;
            System.out.println(new Date(currentTimeMillis));
            ctx.close();
        }
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

注意到这里在 TimeClientHandler 中添加了一个 field, buf 用来缓存从 server 读到的消息, 每次 channelRead 的调用都会填充一部分 ByteBuf, 并且只有在当前 buf 大小已经大于 4 字节时才会从 buf 中获取时间信息

由于 buf 是在 TimeClientHandler 中作为成员变量存在的, 因此其声明周期和 TimeClientHandler 相同, 在 add 方法中初始化, 在 remove 方法中回收

#### second solution (better)

说到底, 这种并非业务逻辑上的问题, 还是尽量不要通过修改 handler 解决, 这里将读取字节的操作放在了 Decoder 中; 在 netty 默认也提供了若干的 Decoder, 这里的 TimeDecoder 就是继承了默认的 ByteToMessageDecoder

```java
// TimeDecoder.java

package icu.buzz.example;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

/**
 * a class used to deal with fragmentation issue
 * ByteToMessageDecoder is a subclass of ChannelInboundHandlerAdapter
 */
public class TimeDecoder extends ByteToMessageDecoder {
    // everytime a new data is received, ByteToMessageDecoder will call decode
    // date will be cumulated into @param: in
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // just return for short counts
        if (in.readableBytes() < 4) return;

        // in will discard the read part of cumulated bytes
       	out.add(in.readBytes(4));
    }
}
```

ByteToMessageDecoder 主要就是为了应对 fragementation 的问题的, 当 client 收到了来自 server 的数据后, 会将消息保存在内部 buffer (in) 中, 同时调用方法 decode()

在确认已经收到了 4 个字节后, 会调用 in.readBytes() 返回 in 中的前 4 个字节, 这里方法 readBytes() 的返回值也是一个 ByteBuf, 相当于和原来的 in 取值相同, 相互独立的 buffer; 只有向 out 添加完 ByteBuf 之后, 才会调用 TimeClientHandler 中的 channelRead 方法

要注意的是, 这个 decoder 也需要在 pipeline 中进行配置

```java
// TimeClient.java

bootstrap.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        // use pipeline to describe operations for SocketChannel
        // add a TimeDecoder to deal with fragmentation issue
        ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
    }
});
```

### POJO

对于业务操作逻辑而言, 直接操作的肯定是对象, 现在借助 netty 中的 decoder 和 encoder 可以让 handler 直接操作对象, 而将对象转为字节的操作放在 decoder 和 encoder 中

在本例中使用 UnixTime 对象对时间进行封装

```java
// UnixTime.java

package icu.buzz.example;

import java.util.Date;

/**
 * wrapper POJO of time
 */
public class UnixTime {
    private final long value;

    public UnixTime() {
        this(System.currentTimeMillis() / 1000L);
    }

    public UnixTime(long time) {
        this.value = time;
    }

    public long value() {
        return this.value;
    }
    @Override
    public String toString() {
        return new Date(value() * 1000L).toString();
    }
}
```

>   本质上就是封装了一个 long 表示时间

这样在 TimeDecoder 中就可以直接向 out 中添加一个对象了

```java
// TimeDecoder.java

@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    // just return for short counts
    if (in.readableBytes() < 4) return;

    // in will discard the read part of cumulated bytes
    out.add(new UnixTime(in.readUnsignedInt()));
}
```

此后在 TimeClientHandler 中的 channelRead 方法中传入的就是一个 UnixTime 对象了

```java
// TimeClientHandler.java

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    UnixTime time = (UnixTime)msg;
    System.out.println(time);
    ctx.close();
}
```

反正是比之前的写法简洁多了, 甚至更好的一个点在于, 这里没有使用 ByteBuf, 也就不需要调用 release 方法进行释放了 ~

除了 client 之外, server 也可以使用类似的方式修改

```java
// TimeServerHandler.java

@Override
public void channelActive(ChannelHandlerContext ctx) {
    // future means IO operation may not be finished yet (do not close channel after writeAndFlush)
    ChannelFuture future = ctx.writeAndFlush(new UnixTime());
    // add a listener to close channel after writeAndFlush
    future.addListener(ChannelFutureListener.CLOSE);
}
```

对于每个连接请求, 直接写入一个 UnixTime 对象即可; 最后就是 encoder 了, 用于 server 将 UnixTime 对象序列化字节

这里的 encoder 有两种实现方式, 一种是通过继承 ChannelOutboundHandlerAdapter, 一种是通过继承 MessageToByteEncoder, 这里选择了后者, 因为更简单

```java
package icu.buzz.example;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

public class TimeEncoder extends MessageToByteEncoder<UnixTime> {
    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, UnixTime unixTime, ByteBuf byteBuf) throws Exception {
        byteBuf.writeInt((int)unixTime.value());
    }
}
```

最后就剩下将 encoder 添加到 pipeline 中了, 要注意的是, 在 netty 中 handler 的顺序是很讲究的, 具体的可以参考 [ChannelPipeline (Netty API Reference (4.1.106.Final))](https://netty.io/4.1/api/io/netty/channel/ChannelPipeline.html)

官方给出了一个示例, 不管是 inbound 还是 outbound 都是针对同一个 pipeline 而言的, 如果是 inbound 就是从 pipeline head 到 tail; 而如果是 outbound 就是从 pipeline 的 tail 到 head; 比如对于如下的 pipeline:

```java
For example, let us assume that we created the following pipeline:

 ChannelPipeline p = ...;
 p.addLast("1", new InboundHandlerA());
 p.addLast("2", new InboundHandlerB());
 p.addLast("3", new OutboundHandlerA());
 p.addLast("4", new OutboundHandlerB());
 p.addLast("5", new InboundOutboundHandlerX());
```

对于 inbound, 数据流向是 1 -> 2 -> 3 -> 4 -> 5; 对于 outbound, 数据流向是 5 -> 4 -> 3 -> 2 -> 1

更为具体的, 实际的 inbound 中, 由于 3 和 4 并没有实现 InboundHandler 接口, 因此实际的数据流向是 1 -> 2 -> 5; 而实际的 outbound 中, 由于 1 和 2 并没有实现 OutboundHandler 接口, 因此实际的数据流向是 5 -> 4 -> 3

所以在 server 端的 pipeline 配置中, encoder 是排在了 handler 之前的

```java
// TimeServer.java

@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
    // use pipeline to describe operations for SocketChannel
    socketChannel.pipeline().addLast(new TimeEncoder(), new TimeServerHandler());
}
```

## structs

### ChannelHandler

netty 使用 ChannelHandler 接口抽象各种 IO 操作, 实际的数据有两个流向, netty 分别使用 ChannelInboundHandler 和 ChannelOutboundHandler 表示

一般而言, 不会直接通过实现接口的方式实现 handler, netty 已经提供了 ChannelInboundHandlerAdapter 和 ChannelOutboundHandlerAdapter 两个适配器实现类

默认的 ChannelInboundHandlerAdapter 和 ChannelOutboundHandlerAdapter 不会对消息进行过多的处理, 而是直接将消息发送给 pipeline 的数据流向中的下一个 handler, 因此通过继承这两个 handler 并重写一种一部分方法是实现数据控制的很好的方式

不过要注意的是由于 ChannelInboundHandlerAdapter 简单的行为 (向后传递), 其不会自动将接收到的 msg 释放掉, 而 SimpleChannelInboundHandler 在 ChannelInboundHandlerAdapter 的基础上, 不仅限制了 msg 的类型, 还会自动释放 msg

>   SimpleChannelInboundHandler 中需要重写的方法为 channelRead0, 而不是 channelRead, 这是因为 SimpleChannelInboundHandler 已经重写了 channelRead, 在其内部会调用 channelRead0 进行逻辑处理, 并且包含了 msg 自动释放的逻辑, 为了使用自动释放的便捷性, 所有业务逻辑建议放在 channelRead0 中

### ChannelPipeline

就把这个 pipeline 当成一个双向链表吧, 在配置 childHandler 的时候, 通过 addLast 和 addFirst 维护了该双向链表

在 netty 中 channel 和 channelpipeline 的关系是一一对应的关系, ChannelPipeline 的双线链表维护的每个元素都是一个 ChannelHandlerContext, 实际的 handler 仅作为 ChannelHandlerContext 的一部分存在

当数据流入时, 会从 head 到 tail 依次调用可用的 handler; 而当数据流出时, 会从 tail 到 head 依次调用可用的 handler; 这也就是上面将 decoder 排列在 handler 之前的原因

### ChannelHandlerContext

除了双向链表中本身的 handler 之外, context 中还保存了 pipeline 和 channel 的信息

### Unpooled

官方的说法: Creates a new ByteBuf by allocating new space or by wrapping or copying existing byte arrays, byte buffers and a string.

>   反正就是和分配 ByteBuf 相关的

*   Allocating a new buffer

    *   `buffer(int)`: allocates a new fixed-capacity heap buffer.
    *   `directBuffer(int)`: allocates a new fixed-capacity direct buffer.

*   Creating a wrapped buffer
    Wrapped buffer is a buffer which is a view of one or more existing byte arrays and byte buffers. Any changes in the content of the original array or buffer will be visible in the wrapped buffer. 

    Various wrapper methods are provided and their name is all wrappedBuffer(). 

*   Creating a copied buffer
    Copied buffer is a deep copy of one or more existing byte arrays, byte buffers or a string. Unlike a wrapped buffer, there's no shared data between the original data and the copied buffer. 

    Various copy methods are provided and their name is all copiedBuffer().

## mini project

### group chat

使用 netty 实现和之前在 NIO 中类似的一个群聊系统, 借助 netty 的封装, 还可以实现用户程序上线下线的监控

netty 的开发是面向传输层协议的, 更进一步的是基于字节的, 而群聊系统是面向字符串的, 因此这里需要使用 StringDecoder 和 StringEncoder 对字节进行编码

这里参考了 StringDecoder 的源码发现, 其在使用的时候需要添加一个 ByteToMessageDecoder -> 其作用主要是用来进行字符串的分割, 表示一段 String 的结束

```java
// StringDecoder.java

/**
 * Decodes a received ByteBuf into a String.  Please
 * note that this decoder must be used with a proper ByteToMessageDecoder
 * such as DelimiterBasedFrameDecoder or LineBasedFrameDecoder
 * if you are using a stream-based transport such as TCP/IP.  A typical setup for a
 * text-based line protocol in a TCP/IP socket would be:
 *
 * ChannelPipeline pipeline = ...;
 *
 * // Decoders
 * pipeline.addLast("frameDecoder", new LineBasedFrameDecoder(80));
 * pipeline.addLast("stringDecoder", new StringDecoder(CharsetUtil.UTF_8));
 *
 * // Encoder
 * pipeline.addLast("stringEncoder", new StringEncoder(CharsetUtil.UTF_8));
 * 
 * and then you can use a String instead of a ByteBuf
 * as a message:
 * 
 * void channelRead(ChannelHandlerContext ctx, String msg) {
 *     ch.write("Did you say '" + msg + "'?\n");
 * }
 * 
 */
```

结合源码中的注释, 在同时使用了 StringDecoder 和一个 ByteToMessageDecoder 之后, 得到的 Message 就可以被认为是一个 String 了

此外群聊还涉及到转发操作, netty 十分贴心的提供了一个 API: ChannelGroup -> 简单来说就是一个 Channel 的集合其主要作用就是在 channel 中广播消息的

```java
// ChannelGroup.java

/**
 * A thread-safe Set that contains open Channels and provides
 * various bulk operations on them.  Using ChannelGroup, you can
 * categorize Channels into a meaningful group (e.g. on a per-service
 * or per-state basis.)  A closed Channel is automatically removed from
 * the collection, so that you don't need to worry about the life cycle of the
 * added Channel.  A Channel can belong to more than one ChannelGroup.
 *
 * Broadcast a message to multiple Channels
 *
 * If you need to broadcast a message to more than one Channel, you can
 * add the Channels associated with the recipients and call ChannelGroup.write(Object):
 * 
 * ChannelGroup recipients = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
 * recipients.add(channelA);
 * recipients.add(channelB);
 * ..
 * recipients.write(Unpooled.copiedBuffer(
 *         "Service will shut down for maintenance in 5 minutes.",
 *         CharsetUtil.UTF_8));
 *
 * Simplify shutdown process with ChannelGroup
 * 
 * If both ServerChannels and non-ServerChannels exist in the
 * same ChannelGroup, any requested I/O operations on the group are
 * performed for the ServerChannels first and then for the others.
 * 
 * This rule is very useful when you shut down a server in one shot:
 *
 * ChannelGroup allChannels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
 *
 * public static void main(String[] args) throws Exception {
 *     ServerBootstrap b = new ServerBootstrap(..);
 *     ...
 *     b.childHandler(new MyHandler());
 *
 *     // Start the server
 *     b.getPipeline().addLast("handler", new MyHandler());
 *     Channel serverChannel = b.bind(..).sync();
 *     allChannels.add(serverChannel);
 *
 *     ... Wait until the shutdown signal reception ...
 *
 *     // Close the serverChannel and then all accepted connections.
 *     allChannels.close().awaitUninterruptibly();
 * }
 *
 * public class MyHandler extends ChannelInboundHandlerAdapter {
 *     @Override
 *     public void channelActive(ChannelHandlerContext ctx) {
 *         // closed on shutdown.
 *         allChannels.add(ctx.channel());
 *         super.channelActive(ctx);
 *     }
 * }
 */
```

将多个 channel 放在一个 channel group 后, 调用 channel group 的 write 方法即可向 group 中所有 channel 完成写操作

ChannelGroup 还具有 graceful shutdown 的特性, 通过将 server channel 本身放在 ChannelGroup 中, 使得在 server 决定关闭连接的时候, 被 ChannelGroup 管理的各个 channel 也会被及时关闭

>   ChannelGroup 更厉害的一个点在于, 如果某个属于 ChannelGroup 的 channel 被人为关闭了, 其并不需要从 ChannelGroup 中移除, netty 会自动将其从 ChannelGroup 中移除

基于这两个点, 其实就可以实现群聊功能了, 首先是 ChatServer

```java
// ChatServer.java

package icu.buzz.example.chat;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.concurrent.GlobalEventExecutor;

import java.nio.charset.StandardCharsets;

public class ChatServer {
    private final int port;
    private final ChannelGroup channelGroup;
    public ChatServer(int port) {
        this.port = port;
        this.channelGroup = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    }

    public void run() {
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) {
                        ChannelPipeline pipeline = socketChannel.pipeline();
                        // as StringDecoder stressed that, StringDecoder must be used with proper ByteToMessageDecoder
                        pipeline.addLast("line based decoder", new LineBasedFrameDecoder(1024))
                                .addLast("String content decoder", new StringDecoder(StandardCharsets.UTF_8))
                                .addLast("String content encoder", new StringEncoder(StandardCharsets.UTF_8))
                                .addLast("Chat server handler", new ChatServerHandler(channelGroup));
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true);
        try {
            ChannelFuture future = bootstrap.bind(this.port).sync();
            // graceful shutdown
            channelGroup.add(future.channel());

            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}

```

还是照旧创建两个 EventLoopGroup -> boss 和 worker, 并在 bootstrap 中绑定关系, 在配置 channel handlers 的时候, 记得配上 StringDecoder/Encoder, 这样自定义的 server handler 就只需要处理 String 类型了

最后为了实现 graceful shutdown, 还将 server channel 保存在了 channel group 中 (尽管在本例中 server 不会主动关闭就是了)

整体上和之前的 Server 没有什么区别, 所有的转发逻辑都放在了 ChatServerHandler 中 (这也说明 netty 确实成功的将连接管理和业务逻辑解耦了)

>   如果不是为了 graceful shutdown, channel group 完全可以在 handler 内部初始化

为了避免主动管理收到的 message, 这里自定义的 handler 继承了 SimpleChannelInboundHandler

```java
// ChatServerHandler.java
package icu.buzz.example.chat;

import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.group.ChannelGroup;
import java.time.format.DateTimeFormatter;

public class ChatServerHandler extends SimpleChannelInboundHandler<String> {
    private final ChannelGroup channelGroup;
    private static final DateTimeFormatter FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    public ChatServerHandler(ChannelGroup channelGroup) {
        this.channelGroup = channelGroup;
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        String timestamp = "[" + FORMATTER.format(java.time.LocalDateTime.now()) + "]";
        String content = timestamp + "client:" + channel.remoteAddress() + " is online\n";
        System.out.print(content);
        channelGroup.writeAndFlush(content);
        channelGroup.add(channel);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        String timestamp = "[" + FORMATTER.format(java.time.LocalDateTime.now()) + "]";
        String content = timestamp + "client:" + channel.remoteAddress() + " is offline\n";
        System.out.print(content);
        channelGroup.writeAndFlush(content, ch -> ch != channel);
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        Channel channel = ctx.channel();
        String timestamp = "[" + FORMATTER.format(java.time.LocalDateTime.now()) + "]";
        String content = timestamp + "client:" + channel.remoteAddress() + " says: " + msg + "\n";
        System.out.print(content);
        channelGroup.writeAndFlush(content, ch -> ch != channel);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close();
    }
}
```

在转发的时候, 为了进行多个用户之间的区分, 这里添加了时间戳和 ip 地址, 用来唯一化用户的发言; 该 handler 也负责了用户上线离线的监控 -> 重写 channelActive 和 channelInactive 方法

注意到 channelGroup 除了简单的 write 方法之外, 还提供了一个携带了 ChannelMatcher 的方法, 该接口用来对 channel 进行过滤, 只有满足条件的 channel 才会被写入内容

剩下的就是客户端了, 这个就很好实现了

```java
// ChatClient.java

package icu.buzz.example.chat;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.nio.charset.StandardCharsets;
import java.util.Scanner;

public class ChatClient {
    private String host;
    private int port;

    public ChatClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void run() {
        EventLoopGroup worker = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(worker)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast("line based decoder", new LineBasedFrameDecoder(1024))
                                .addLast("String content decoder", new StringDecoder(StandardCharsets.UTF_8))
                                .addLast("String content encoder", new StringEncoder(StandardCharsets.UTF_8))
                                .addLast("Chat client handler", new ChatClientHandler());
                    }
                })
                .option(ChannelOption.SO_KEEPALIVE, true);
        try {
            ChannelFuture future = bootstrap.connect(this.host, this.port).sync();
            Scanner in = new Scanner(System.in);
            while (in.hasNextLine()) {
                String line = in.nextLine() + "\n";
                future.channel().writeAndFlush(line);
            }
            // shutdown after eof
            future.channel().close().sync();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            worker.shutdownGracefully();
        }
    }
}
```

相比之前的 client, 额外添加了 Decoder/Encoder, 同时使用 EOF (ctrl + d) 表示 client 主动断开连接

```java
// ChatClientHandler.java

package icu.buzz.example.chat;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

public class ChatClientHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println(msg);
    }
}
```

写操作放在了 client 中本身, 那么剩下的就是在 handler 进行读了 ...

### idle check

一般而言网络连接都是带有有效性检测的, 超时后及时断开连接, netty 通过 handler 的方式集成了这个功能 -> IdleStateHandler

```java
// IdleStateHandler.java

/**
 * Triggers an IdleStateEvent when a Channel has not performed
 * read, write, or both operation for a while.
 *
 * Supported idle states
 * readerIdleTime: an IdleStateEvent whose state is IdleState.READER_IDLE will
 * be triggered when no read was performed for the specified period of time. Specify 0 to disable.
 * 
 * writerIdleTime: an IdleStateEvent whose state is IdleState.WRITER_IDLE will 
 * be triggered when no write was performed for the specified period of time. Specify 0 to disable.
 *  
 * allIdleTime: an IdleStateEvent whose state is IdleState.ALL_IDLE will 
 * be triggered when neither read nor write was performed for the specified period of time. Specify 0 to disable.
 *  
 * // An example that sends a ping message when there is no outbound traffic
 * // for 30 seconds.  The connection is closed when there is no inbound traffic
 * // for 60 seconds.
 *
 * public class MyChannelInitializer extends ChannelInitializer<Channel> {
 *     @Override
 *     public void initChannel(Channel channel) {
 *         channel.pipeline().addLast("idleStateHandler", new IdleStateHandler(60, 30, 0));
 *         channel.pipeline().addLast("myHandler", new MyHandler());
 *     }
 * }
 *
 * // Handler should handle the IdleStateEvent triggered by IdleStateHandler.
 * public class MyHandler extends ChannelDuplexHandler {
 *     @Override
 *     public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
 *         if (evt instanceof IdleStateEvent) {
 *             IdleStateEvent e = (IdleStateEvent) evt;
 *             if (e.state() == IdleState.READER_IDLE) {
 *                 ctx.close();
 *             } else if (e.state() == IdleState.WRITER_IDLE) {
 *                 ctx.writeAndFlush(new PingMessage());
 *             }
 *         }
 *     }
 * }
 *
 * ServerBootstrap bootstrap = ...;
 * ...
 * bootstrap.childHandler(new MyChannelInitializer());
 * ...
 */
```

>   netty 官方的注释很贴心, 还给出了示例代码

简单来说 IdleStateHandler 会在没有 IO 操作的一段时间后触发事件, 为了让该事件可以让用户自定义的 handler 处理, 在 pipeline 中需要放在自定义 handler 之前

一般的网络相关的项目中, 都需要通过心跳包进行链接的检测, 在注释给出的示例中, 如果 server 在 30s 的时间内没有向 client 发送任何数据 (write idle), 就会将其发送 ping message; 而如果在 60s 内没有收到来自 client 的任何相应, 则会直接断开链接

### websocket

一般而言不需要重头开发一个应用层的协议, 比如上面的聊天程序可以被修改为基于 websocket

```java
package icu.buzz.chat;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;

public class WebSocketServer {
    private final String endpoint;
    private final int port;

    public WebSocketServer(String endpoint, int port) {
        this.endpoint = endpoint;
        this.port = port;
    }

    public void run() {
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast("http coder/decoder", new HttpServerCodec())
                                .addLast("aggregator", new HttpObjectAggregator(65535))
                                .addLast("WebSocket protocol handler", new WebSocketServerProtocolHandler(endpoint))
                                .addLast("self defined handler", new WebSocketServerHandler());
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true);
        try {
            ChannelFuture future = bootstrap.bind(this.port).sync();
            // close the server
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            worker.shutdownGracefully();
            boss.shutdownGracefully();
        }
    }
}
```

在经过 WebSocketServerProtocolHandler 之后, 消息会以 WebSocketFrame 的形式呈递给 WebSocketServerHandler

## kryo

很多时候 netty 交互的并不直接是文本数据, 可能是某些序列化后的对象, netty 默认提供了 ObjectDecoder/ObjectEncoder 用来将对象解码/编码 -> 本质上使用 jdk 原生序列化对象的 api

[EsotericSoftware/kryo: Java binary serialization and cloning: fast, efficient, automatic](https://github.com/EsotericSoftware/kryo) 是一个开源的序列化库, 如果没有多语言环境需求的话就用这个吧

>   dubbo 已经准备使用 kryo 替换 Hessian 了 -> [Kryo 和 FST 序列化 | Apache Dubbo](https://cn.dubbo.apache.org/zh-cn/docsv2.7/user/serialization/)
>
>   如果有跨平台需求的话可以用 google 的 [protocolbuffers/protobuf: Protocol Buffers - Google's data interchange format](https://github.com/protocolbuffers/protobuf)

官方给出了一个 quick start, 十分好用

```java
// User.java

package icu.buzz;

public class User {
    private String name;
    private String password;

    public User() {
    }

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }

    public String getName() {
        return name;
    }

    public String getPassword() {
        return password;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}

// Main.java

package icu.buzz;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;

public class Main {
    public static void main(String[] args) throws FileNotFoundException {
        Kryo kryo = new Kryo();
        kryo.register(User.class);

        User u = new User("buzz", "buzz");

        Output output = new Output(new FileOutputStream("user.bin"));
        kryo.writeObject(output, u);
        output.close();

        Input input = new Input(new FileInputStream("user.bin"));
        User uu = kryo.readObject(input, User.class);
        input.close();

        System.out.println(uu.getName());
        System.out.println(uu.getPassword());
    }
}
```

>   将对象序列化并写入到文件 "user.bin" 中; 并从该文件中读取数据并返回一个对象

## RPC

借助 netty 框架, 使用 kryo 进行序列化, 实现 RPC

RPC 的关键在于代理模式, 当 client 端进行函数调用时, 通过代理模式, 中断方法调用, 而将方法名、参数等信息封装好, 发送给 server 端进行方法调用, 并借助 java 的 futrue 特性阻塞当前线程, 等待 RPC 的结果返回

### RPC object

这里使用 RpcRequest 和 RpcResponse 分别表示 RPC 请求和响应

```java
// RpcRequest.java

package icu.buzz.rpc;

public class RpcRequest {
    private String serviceName;
    private String methodName;
    private Object[] params;

    // reserved for kryo
    public RpcRequest() {
    }

    public RpcRequest(String serviceName, String methodName, Object[] params) {
        this.serviceName = serviceName;
        this.methodName = methodName;
        this.params = params;
    }

    public String getServiceName() {
        return serviceName;
    }

    public void setServiceName(String serviceName) {
        this.serviceName = serviceName;
    }

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }

    public Object[] getParams() {
        return params;
    }

    public void setParams(Object[] params) {
        this.params = params;
    }
}
```

在 RPC 中 client 和 server 双方需要共享接口, 在 client 端调用这个接口, 而在 server 端实现这个接口; 为了 RPC 的通用性, 这里需要封装对应的接口名 (serviceName), 调用的方法名 (methodName), 调用方法的参数列表 (params)

```java
// RpcResponse.java

package icu.buzz.rpc;

public class RpcResponse {
    private Object rst;
    private String msg;
    public RpcResponse() {
    }

    public RpcResponse(Object rst, String msg) {
        this.rst = rst;
        this.msg = msg;
    }

    public Object getRst() {
        return rst;
    }

    public void setRst(Object rst) {
        this.rst = rst;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

RpcResponse 中封装了方法调用的结果 (rst), 而这种涉及到网络连接的应用通常是不可靠的, 因此这里还封装了 msg 用来指示方法调用的情况

值得注意的是, 这里显示的添加了无参构造方法, 这里主要是因为 kryo 默认需要通过调用无参构造方法才能进行实例对象的创建 (当然这个是可以配置的 ([EsotericSoftware/kryo: Java binary serialization and cloning: fast, efficient, automatic](https://github.com/EsotericSoftware/kryo?tab=readme-ov-file#object-creation) 我懒得配置就是了)

### coder/decoder

这部分涉及到 kryo 的配置, 对于 server 而言, 需要接收一个 RpcRequest 并返回一个 RpcResponse; 而对于 client 而言, 需要发送一个 RpcRequest 并接收一个 RpcResponse

因此 server 端和 client 端的 coder/decoder 是不同的; 这里自定义了 RpcCodec 继承了类 ByteToMessageCodec 并在方法 decode 和 encode 方法中分别进行反序列化和序列化操作

```java
// client -> RpcCodec.java

package icu.buzz.rpc;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Input;
import com.esotericsoftware.kryo.io.Output;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageCodec;

import java.util.List;

public class RpcCodec extends ByteToMessageCodec<RpcRequest> {
    private final Kryo kryo;

    public RpcCodec() {
        this.kryo = new Kryo();
        kryo.register(RpcRequest.class);
        kryo.register(RpcResponse.class);
        kryo.register(Object[].class);
    }

    @Override
    protected void encode(ChannelHandlerContext ctx, RpcRequest msg, ByteBuf out) {
        // try with resources will close the output automatically
        // create a kryo output buffer with size 1024, maximum size of the buffer is -1 (on limit)
        try (Output output = new Output(1024, -1)) {
            // write object into kryo output
            kryo.writeObject(output, msg);
            // get output size
            int len = output.position();
            // write object length into ByteBuf
            out.writeInt(len);
            // write object self into ByteBuf
            out.writeBytes(output.getBuffer(), 0, len);
        }
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        if (in.readableBytes() < 4) {
            return;
        }
        // mark current position -> size of Object
        in.markReaderIndex();
        int len = in.readInt();
        if (in.readableBytes() < len) {
            // reset to the marked position
            in.resetReaderIndex();
            return;
        }

        byte[] bytes = new byte[len];
        in.readBytes(bytes);

        // try with resources will close the input automatically
        try (Input input = new Input(bytes)) {
            RpcResponse response = kryo.readObject(input, RpcResponse.class);
            out.add(response);
        }
    }
}
```

在使用 kryo 进行对象的序列化之前, 首先需要进行类型的注册, 具体的可以参考官方文档

这里的 RpcCodec 是 client 端的, server 端的实现是类似的, 但要注意上面提到的不同

序列化的格式为: 对象大小 (4 Bytes) + 对象本身

由于这里并不是将对象保存在文件当中, 因此这里的 encode 方法中创建的 Output 其实是在内存中开辟了一个字节数组, 并将对象临时序列化到内存中, 随后调用方法 position 获取对象大小, 然后依次将大小和对象本身添加到 out 中

decode 方法在反序列化的时候, 首先需要获取对象的大小, 由于这里使用 4 个字节表示对象的大小, 因此如果可读的字节个数不足 4 个时将直接返回; 在获取到对象的大小后, 后续可读的字节数可能不足一个对象的大小, 此时需要重置, 在进行对象读取的时候, 可读的字节数一定达到了一个对象的大小

### client

上面也提到了, RPC 的关键在于代理模式, 使得程序可以中断方法的调用, 并通过网络调用的方式取代直接的方法调用

client 和 server 需要共享接口, 这里的 client 通过调用一个静态方法完成对象的获取, 并执行方法调用

```java
// client.java

package icu.buzz;

import icu.buzz.rpc.ServiceFactory;

public class Client {
    public static void main(String[] args) {
        // rpc client
        Service service = ServiceFactory.getService(Service.class);
        System.out.println(service.echo("hello"));
        System.out.println(service.echo("buzz"));
    }
}
```

可以预见的是, 本质上 getService 方法就是一个 Proxy.newProxyInstance -> 就是一个动态代理

```java
// ServiceFactory.java

package icu.buzz.rpc;

import java.lang.reflect.Proxy;

public class ServiceFactory {

    public static <T> T getService(Class<T> klass) {
        return (T) Proxy.newProxyInstance(klass.getClassLoader(), new Class<?>[]{klass}, new RpcHandler());
    }
}
```

类 RpcHandler 实现了接口 InvocationHandler, 表示实际中断方法调用后的处理程序

```java
// RpcHandler.java

package icu.buzz.rpc;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class RpcHandler implements InvocationHandler {
    private static final String HOST = "localhost";
    private static final int PORT = 8888;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        Consumer consumer = new Consumer(HOST, PORT);
        try {
            consumer.init();
            RpcRequest request = new RpcRequest(method.getDeclaringClass().getName(), method.getName(), args);
            RpcResponse response = consumer.invoke(request);
            if (response == null) return null;
            return response.getRst();
        } catch (InterruptedException e) {
            return null;
        } finally {
            consumer.release();
        }
    }
}
```

注意到这里就是通过 Consumer 对象创建了一个到 server 的连接, 同时调用方法 invoke 进行 RPC, 最后将结果返回

>   还是挺简单的

Consumer 也没什么的, 和之前看到的 netty client 类似

```java
// Consumer.java

package icu.buzz.rpc;

import com.esotericsoftware.kryo.Kryo;
import com.esotericsoftware.kryo.io.Output;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class Consumer {
    private final String host;

    private final int port;

    private EventLoopGroup worker;
    private Channel channel;

    public Consumer(String host,int port) {
        this.host = host;
        this.port = port;
    }

    public void init() throws InterruptedException {
        this.worker = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(worker)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast("rpc coder/decoder", new RpcCodec())
                                .addLast("Consumer Handler", new ConsumerHandler());
                    }
                })
                .option(ChannelOption.SO_KEEPALIVE, true);
        this.channel = bootstrap.connect(host, port).sync().channel();
    }

    public void release() {
        try {
            if (channel != null && channel.isOpen()) this.channel.close().sync();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            this.worker.shutdownGracefully();
        }
    }

    public RpcResponse invoke(RpcRequest request) {
        CompletableFuture<RpcResponse> future = new CompletableFuture<>();
        channel.pipeline().get(ConsumerHandler.class).setFuture(future);
        channel.writeAndFlush(request);
        try {
            return future.get();
        } catch (InterruptedException | ExecutionException e) {
            return null;
        }
    }
}
```

和之前最大的区别在于, 这里的 client 需要等待方法 invoke 的调用才会向 server 发送数据, 同时写数据的操作需要维护一个 channel 引用, 这里将 channel 的作用域提升为一个成员变量

由于 RPC 需要等待 server 端的处理, 这里出于简便, 就然 client 保持阻塞, 直到收到了一个响应; 这里使用了 future 特性, 在进行方法调用之前, 为 handler 设置一个 future, 随后便一直阻塞在该 future 上

可以预见的是, handler 的 channelRead 方法其实就是给这个 future 设置了一个返回值

```java
// ConsumerHandler.java

package icu.buzz.rpc;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

import java.util.concurrent.CompletableFuture;

public class ConsumerHandler extends SimpleChannelInboundHandler<RpcResponse> {

    private CompletableFuture<RpcResponse> future;

    public void setFuture(CompletableFuture<RpcResponse> future) {
        this.future = future;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcResponse msg) throws Exception {
        future.complete(msg);
    }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        future.completeExceptionally(cause);
        ctx.close();
    }
}
```

### server

server 端的处理相对就简单的多了, 毕竟不再涉及到动态代理以及 future 特性这些东西了

```java
// Provider.java

package icu.buzz.rpc;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

public class Provider {
    private final int port;

    public Provider(int port) {
        this.port = port;
    }

    public void run() {
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast("rpc coder/decoder", new RpcCodec())
                                .addLast("Service Provider", new ProviderHandler());
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true);
        try {
            ChannelFuture future = bootstrap.bind(this.port).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}
```

注意到这里 server 的配置和之前的 netty server 不能说完全一致吧, 只能说毫无区别

```java
// ProviderHandler.java

package icu.buzz.rpc;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class ProviderHandler extends SimpleChannelInboundHandler<RpcRequest> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcRequest msg) {
        RpcResponse response = new RpcResponse(null);
        try {
            // get class
            Class<?> klass = Class.forName(msg.getServiceName() + "Impl");
            // get method
            Object[] params = msg.getParams();
            Class<?>[] paramTypes = new Class[params.length];
            for (int i = 0; i < params.length; i++) paramTypes[i] = params[i].getClass();
            Method method = klass.getMethod(msg.getMethodName(), paramTypes);
            // new an instance
            Object instance = klass.getDeclaredConstructor().newInstance();

            Object result = method.invoke(instance, params);

            response.setRst(result);
            response.setMsg("all good");
        } catch (ClassNotFoundException e) {
            response.setMsg("service has not been provided");
        } catch (NoSuchMethodException | SecurityException e) {
            response.setMsg("function has not been provided");
        } catch (InvocationTargetException | InstantiationException | IllegalAccessException e) {
            throw new RuntimeException(e);
        } finally {
            ctx.writeAndFlush(response);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close();
    }
}
```

注意到 handler 的实现, 因为 RpcRequest 中仅仅包含了类名和方法名, 为了在 server 端进行方法调用, 这里需要使用反射

这里认为每个在 RPC 中提供的服务 (接口), 其实现类在名字上都添加了一个 Impl, 并进行类型的搜索; 然后需要根据参数类型和方法名获取对应的方法; 在通过反射创建对象后, 调用对应的方法, 并将结果封装到 RpcResponse 中返回给 client

# Server IO model

>   [Essential Technologies for Java Developers: I/O and Netty](https://www.alibabacloud.com/blog/essential-technologies-for-java-developers-io-and-netty_597367)

## traditional IO model

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/06/24/21:15:02:traditional_io_model.png)

server 对于每个来自 client 的 request 使用独立的一个进程 (线程) 处理, 这个进程 (线程) 需要负责: 接受 -> 解码 -> 计算 -> 编码 -> 传输; 每个进程都需要处理完整的流程

request 的数目越多 server 创建的进程 (线程) 数目越多, 在大量请求的场景中, 性能显著恶化; 一般而言可以通过线程池限制 server 创建进程的数目, 缓解这个问题

但是这个 model 存在一个很大的问题, 即如果 request 的 compute 时间过长, 那么整个 server 将不能处理任何来自 client 的请求

## reactor model

基于事件的 IO 处理模型, 主要由两部分组成:

*   reactor: 一个独立的线程, 负责监听事件, 并调用适当的 handler 处理事件
*   handler: 处理 IO 请求, 可能是多线程的

reactor model 本身也可以分为多种类型

### single-threaded

one reactor + single thread => redis

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/06/24/21:26:44:single_thread_reactor_model.png)

reactor 使用 selector 监测 connection event 与 receive event; reactor 使用 acceptor 作为 connection event 的 handler, 使用更为一般的 handler 处理 read 请求

>   acceptor 本身也只是创建一个普通的 handler 而已

值得注意的是所有的 handle 都是按照单线程执行的 (IO multiplexing)

### multi-threaded

one reactor + multi thread

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/06/24/21:32:15:multi_thread_reactor_model.png)

注意到和上面最大的区别在于, handler 仅仅负责 read request 与 send response, 而 server 本身会使用线程池对 request 进行处理

在该模式下, 本质上还是使用线程池进行计算, 和 traditional model 相比, 可以在大量 request 下仍能够处理 client 请求, 但也就止于此了 (还是可能被恶意 request block 掉)

### primary-secondary threaded

multi reactor + multi thread 

![](https://cdn.jsdelivr.net/gh/buzzxI/img@latest/img/24/06/24/21:40:05:primary_secondary_reactor_model.png)

main reactor monitor connection event, sub reactor create handler to handle subsequent event



















































