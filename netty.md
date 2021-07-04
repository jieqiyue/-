### netty的线程模型

Netty NIO分配EventLoop模型

![image-20210225180755334](../../TyporaImages/image-20210225180755334.png)

一个EventLoop由一个线程和一个Selector组成。由图可以知道，在NIO模式下，一个Channel只会被分配给一个EventLoop。而一个EventLoop可能会被分配给多个Channel来负责上面的事件。

### Netty中的handler()和childHandler()、childOption()和option有什么区别

https://blog.csdn.net/Lemon_MY/article/details/107220554 

### handler之间的传递

1. 继承了 SimpleChannelInboundHandler<T> 的 handler 需要显式的调用ctx.fireChannelRead(messageRequestPacket); 来进行往下个handler的传递。如果你没有调用，则下一个handler不会被调用到。

   如果一个handler继承了SimpleChannelInboundHandler，那么它是根据后面的泛型T来判断自己是否能够处理该msg的。如果发现自己不能够处理，netty将会自动的将它传递到下一个handler去进行处理。

   通过实验得出，当一个继承了SimpleChannelInboundHandler的类处理了之后（它发现自己能够强转为 T ，自己有资格处理它），那么后续如果还有一个类也能够处理这个T的话，那么还是需要自己去显式的调用ctx.fireChannelRead(messageRequestPacket);的。不然的话，后面的类是处理不到的。

   如下图：![image-20210225201316897](../../TyporaImages/image-20210225201316897-1617197367153.png)

   ![image-20210225201400762](../../TyporaImages/image-20210225201400762.png)![image-20210225201412951](../../TyporaImages/image-20210225201412951.png)

2. 继承 ChannelInboundHandlerAdapter 也需要显式的调用往下传递的。继承这个类，代表这个 handler 可以处理所有类型的消息。

3. 可以通过调用`   ctx.channel().close();` 来阻止继续往下面的 handler 调用。

ChannelInboundHandlerAdapter 中的channelActive会在建立连接的时候被调用，如果有两个handler都有这个channelActive那么如果前面那个handler不调用`super.channelActive(ctx);`的话，后面的是接收不到这个信息的。

### ChannelGroup 的聚合发送功能

这里简单介绍一下 `ChannelGroup`：它可以把多个 chanel 的操作聚合在一起，可以往它里面添加删除 channel，可以进行 channel 的批量读写，关闭等操作

### ByteToMessageDecoder

通常情况下，无论我们是在客户端还是服务端，当我们收到数据之后，首先要做的事情就是把二进制数据转换到我们的一个 Java 对象，所以 Netty 很贴心地写了一个父类，来专门做这个事情。这个类就是ByteToMessageDecoder。为什么需要这样呢？因为在通信的时候，底层传输的数据都是二进制数据。也就是Byte。那么，当我们只是简单地传递一些字符串的话，可能是像下面这样的：![image-20210226124450254](../../TyporaImages/image-20210226124450254.png)

![image-20210226124552604](../../TyporaImages/image-20210226124552604.png)

这样的话，只是简单的一些字节。那么，如果我们要传递一个Java对象呢，此时就需要通过序列化的手段（Jackson，fastjson，等）将一个Java对象转换为字节。然后在对端还需要对这个字节数组进行反序列化。

### 粘包与拆包

尽管我们在应用层面使用了 Netty，但是对于操作系统来说，只认 TCP 协议，尽管我们的应用层是按照 ByteBuf 为 单位来发送数据，但是到了底层操作系统仍然是按照字节流发送数据，因此，数据到了服务端，也是按照字节流的方式读入，然后到了 Netty 应用层面，重新拼装成 ByteBuf，而这里的 ByteBuf 与客户端按顺序发送的 ByteBuf 可能是不对等的。因此，我们需要在客户端根据自定义协议来组装我们应用层的数据包，然后在服务端根据我们的应用层的协议来组装数据包，这个过程通常在服务端称为拆包，而在客户端称为粘包。

拆包和粘包是相对的，一端粘了包，另外一端就需要将粘过的包拆开，举个栗子，发送端将三个数据包粘成两个 TCP 数据包发送到接收端，接收端就需要根据应用协议将两个数据包重新组装成三个数据包。

netty中自带了一些拆包器。比如：LengthFieldBasedFrameDecoder

```java
public class Spliter extends LengthFieldBasedFrameDecoder {
    private static final int LENGTH_FIELD_OFFSET = 7;
    private static final int LENGTH_FIELD_LENGTH = 4;

    public Spliter() {
        super(Integer.MAX_VALUE, LENGTH_FIELD_OFFSET, LENGTH_FIELD_LENGTH);
    }
	// 这个decode是为了判断ByteBuf开头的魔数是否符合规范
    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        if (in.getInt(in.readerIndex()) != PacketCodeC.MAGIC_NUMBER) {
            ctx.channel().close();
            return null;
        }

        return super.decode(ctx, in);
    }
}
```

这个类先定义好，当然netty原来的类是LengthFieldBasedFrameDecoder，在上面这个自定义的类中，主要是为了判断是否是本server能够支持的数据包。最重要的是要看看构造函数。此时还需要注意一点，虽然构造函数中指出了偏移量和包体长度。但是拆包后，我们得到的ByteBuf并没有将开头的协议头给去掉。在后续的Decode中还是需要跳过一些字段的。

```java 
 public Packet decode(ByteBuf byteBuf) {
        // 跳过 magic number
        byteBuf.skipBytes(4);

        // 跳过版本号
        byteBuf.skipBytes(1);

        // 序列化算法
        byte serializeAlgorithm = byteBuf.readByte();

        // 指令
        byte command = byteBuf.readByte();

        // 数据包长度
        int length = byteBuf.readInt();

        byte[] bytes = new byte[length];
        byteBuf.readBytes(bytes);

        Class<? extends Packet> requestType = getRequestType(command);
        Serializer serializer = getSerializer(serializeAlgorithm);

        if (requestType != null && serializer != null) {
            return serializer.deserialize(requestType, bytes);
        }

        return null;
    }
```

### 空闲检测

```java
public class IMIdleStateHandler extends IdleStateHandler {

    private static final int READER_IDLE_TIME = 15;

    public IMIdleStateHandler() {
        super(READER_IDLE_TIME, 0, 0, TimeUnit.SECONDS);
    }

    @Override
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) {
        System.out.println(READER_IDLE_TIME + "秒内未读到数据，关闭连接");
        ctx.channel().close();
    }
}

```

在netty中提供各类一个空闲检测的类。就是IdleStateHandler。在构造函数中可以看到，有四个参数。其中第一个表示读空闲时间，指的是在这段时间内如果没有数据读到，就表示连接假死；第二个是写空闲时间，指的是 在这段时间如果没有写数据，就表示连接假死；第三个参数是读写空闲时间，表示在这段时间内如果没有产生数据读或者写，就表示连接假死。写空闲和读写空闲为0，表示我们不关心者两类条件；最后一个参数表示时间单位。在我们的例子中，表示的是：如果 15 秒内没有读到数据，就表示连接假死。连接假死之后会回调 `channelIdle()` 方法，我们这个方法里面打印消息，并手动关闭连接。

#### 客户端定期发送数据包给服务端

```java
public class HeartBeatTimerHandler extends ChannelInboundHandlerAdapter {

    private static final int HEARTBEAT_INTERVAL = 5;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        scheduleSendHeartBeat(ctx);

        super.channelActive(ctx);
    }

    private void scheduleSendHeartBeat(ChannelHandlerContext ctx) {
        ctx.executor().schedule(() -> {

            if (ctx.channel().isActive()) {
                ctx.writeAndFlush(new HeartBeatRequestPacket());
                scheduleSendHeartBeat(ctx);
            }

        }, HEARTBEAT_INTERVAL, TimeUnit.SECONDS);
    }
}
```

`ctx.executor()` 返回的是当前的 channel 绑定的 NIO 线程。NIO 线程有一个方法，`schedule()`，类似 jdk 的延时任务机制，可以隔一段时间之后执行一个任务，而我们这边是实现了每隔 5 秒，向服务端发送一个心跳数据包，这个时间段通常要比服务端的空闲检测时间的一半要短一些，我们这里直接定义为空闲检测时间的三分之一，主要是为了排除公网偶发的秒级抖动。

### 事件传播源

`ctx.writeAndFlush()` 事件传播路径

`ctx.writeAndFlush()` 是从 pipeline 链中的当前节点开始往前找到第一个 outBound 类型的 handler 把对象往前进行传播，如果这个对象确认不需要经过其他 outBound 类型的 handler 处理，就使用这个方法。

`ctx.channel().writeAndFlush()` 事件传播路径

`ctx.channel().writeAndFlush()` 是从 pipeline 链中的最后一个 outBound 类型的 handler 开始，把对象往前进行传播，如果你确认当前创建的对象需要经过后面的 outBound 类型的 handler，那么就调用此方法。

### 服务端的启动流程和参数配置

=======================================================================================

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/616ebe9dc6a649a48493eb1a1c3a5aa1~tplv-k3u1fbpfcp-zoom-1.image?imageslim)

### 处理器链的实现

自定义一个类去继承`ChannelInboundHandlerAdapter` 。然后可以重写`channelActive` 方法。和 `channelRead` 方法。分别实现在建立连接后调用的方法和在接收到数据的时候调用的方法。

### BtyeBuf 



### 心跳机制

https://juejin.cn/book/6844733738119593991/section/6844733738291576846

https://segmentfault.com/a/1190000006931568

https://blog.csdn.net/xiaoduanayu/article/details/78386508?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&dist_request_id=1328767.57511.16175997267242071&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control

TCP keepalive指的是TCP保活计时器（keepalive timer）。设想有这样的情况：客户已主动与服务器建立了TCP连接。但后来客户端的主机突然出故障。显然，服务器以后就不能再收到客户发来的数据。因此，应当有措施使服务器不要再白白等待下去。这就是使用保活计时器。服务器每收到一次客户的数据，就重新设置保活计时器，时间的设置通常是两小时。若两小时没有收到客户的数据，服务器就发送一个探测报文段，以后则每隔75秒发送一次。若一连发送10个探测报文段后仍无客户的响应，服务器就认为客户端出了故障，接着就关闭这个连接。

 ——摘自谢希仁《计算机网络》

> 为什么tcp层面有keepalive机制，还需要在应用层去实现心跳机制呢？

因为tcp层面的keepalive机制是说，服务器在两个小时之内没有收到客户端的数据包才回去发送心跳连接。那么，两个小时太久了。在Linux中可以使用以下命令查看配置：

```shell
# cat /proc/sys/net/ipv4/tcp_keepalive_time  7200 当keepalive启用的时候，TCP发送keepalive消息的频度。缺省是2小时
# cat /proc/sys/net/ipv4/tcp_keepalive_intvl  75  当探测没有确认时，重新发送探测的频度。缺省是75秒
# cat /proc/sys/net/ipv4/tcp_keepalive_probes  9  探测尝试的次数。如果第1次探测包就收到响应了，则后8次的不再发
```

所以，需要在应用层，也定时的发送一些心跳包来保证这个连接的双方是正常的。

而且，对于服务端来说，因为每条连接都会耗费 cpu 和内存资源，大量假死的连接会逐渐耗光服务器的资源，最终导致性能逐渐下降，程序奔溃。对于客户端来说，连接假死会造成发送数据超时，影响用户体验。那么在产生这种假死的状态的时候，也需要应用层发送心跳包来进行检测。并进行处理。

解决方法：

对于netty来说，已经有对应的类了。Netty 自带的 `IdleStateHandler` 就可以实现这个功能。

```java
public class IMIdleStateHandler extends IdleStateHandler {

    private static final int READER_IDLE_TIME = 15;

    public IMIdleStateHandler() {
        super(READER_IDLE_TIME, 0, 0, TimeUnit.SECONDS);
    }

    @Override
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) {
        System.out.println(READER_IDLE_TIME + "秒内未读到数据，关闭连接");
        ctx.channel().close();
    }
}
```

1. 首先，我们观察一下 `IMIdleStateHandler` 的构造函数，他调用父类 `IdleStateHandler` 的构造函数，有四个参数，其中第一个表示读空闲时间，指的是在这段时间内如果没有数据读到，就表示连接假死；第二个是写空闲时间，指的是 在这段时间如果没有写数据，就表示连接假死；第三个参数是读写空闲时间，表示在这段时间内如果没有产生数据读或者写，就表示连接假死。写空闲和读写空闲为0，表示我们不关心者两类条件；最后一个参数表示时间单位。在我们的例子中，表示的是：如果 15 秒内没有读到数据，就表示连接假死。
2. 连接假死之后会回调 `channelIdle()` 方法，我们这个方法里面打印消息，并手动关闭连接。

接下来，我们把这个 handler 插入到服务端 pipeline 的最前面。服务端在一段时间内没有收到数据的话，就会主动断开连接，避免了服务端的资源一直被占用。这里是说服务端的代码实现。那么在客户端也需要定期的去发送心跳包，这个可以使用与客户端NIO线程设置一个定时任务来实现。就比如说，客户端每隔五秒就发送一个数据包。那么，服务端在15秒内没有接收到数据的话，那么就会断开连接。

同样的，客户端也需要能够主动断开连接，如果是服务端挂了呢？那么客户端也需要在15秒内没有接收到数据的话，就断开连接。这个实现也很简单。在服务端收到数据的时候，也发送给客户端一个响应报文就行了。

客户端代码如下：

```java
public class HeartBeatTimerHandler extends ChannelInboundHandlerAdapter {

    private static final int HEARTBEAT_INTERVAL = 5;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        scheduleSendHeartBeat(ctx);
        System.out.println("client的心跳机制！");
        //super.channelActive(ctx);
    }

    private void scheduleSendHeartBeat(ChannelHandlerContext ctx) {
        ctx.executor().schedule(() -> {

            if (ctx.channel().isActive()) {
                System.out.println("每隔五秒就发送一次");
                ctx.writeAndFlush(new HeartBeatRequestPacket());
                scheduleSendHeartBeat(ctx);
            }

        }, HEARTBEAT_INTERVAL, TimeUnit.SECONDS);
    }
}
```

![image-20210509122350520](../../TyporaImages/image-20210509122350520.png)

