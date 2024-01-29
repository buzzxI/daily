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











