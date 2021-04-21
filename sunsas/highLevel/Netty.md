# 4.1 Netty

> 作者：SunSAS
>
> **介绍：** SunSAS是SunSAS

## 4.1.2 Netty源码

### Netty源码（一）启动源码分析

随便建一个boot项目，引入netty依赖

```
<!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.42.Final</version>
</dependency>
```

#### 1.新建一个启动服务器类

```java
package com.sunsas.spingtest.test;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.sctp.nio.NioSctpServerChannel;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.FixedLengthFrameDecoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;

/**
 * @author: yuwenping
 * @date: 2021/2/22 10:44
 * @Description:
 */
public class EchoServer {
    public void startEchoServer(int port) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workGroup)
                    .channel(NioSctpServerChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))//设置ServerSocketChannel 对应的 Handler
                    .childHandler(new ChannelInitializer<SocketChannel>() {// 设置 SocketChannel 对应的 Handler
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new FixedLengthFrameDecoder(10));
                            // ResponseSampleEncoder 用于将服务端的处理结果进行编码
                            socketChannel.pipeline().addLast(new ResponseSampleEncoder());
                            //RequestSampleHandler 主要负责客户端的数据处理，并通过调用 ctx.channel().writeAndFlush 向客户端返回 ResponseSample 对象，其中包含返回码、响应数据以及时间戳。
                            socketChannel.pipeline().addLast(new RequestSampleHandler());
                        }
                    });
            // bind() 才是真正进行服务器端口绑定和启动的入口，sync() 表示阻塞等待服务器启动完成。
            ChannelFuture f = b.bind(port).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
}
```

`ResponseSampleEncoder`是自定义的编码器，
`RequestSampleHandler`是自定义的客户端的数据处理器，
`ResponseSample` 是自定义的返回类。

```java
public class ResponseSampleEncoder extends MessageToByteEncoder<ResponseSample>{

    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, ResponseSample msg, ByteBuf byteBuf) throws Exception {
        if(msg != null){
            byteBuf.writeBytes(msg.getCode().getBytes());
            byteBuf.writeBytes(msg.getData().getBytes());
            byteBuf.writeLong(msg.getTimestamp());
        }
    }
}
```

```java
public class RequestSampleHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg){
        String data = ((ByteBuf) msg).toString(CharsetUtil.UTF_8);
        ResponseSample responseSample = new ResponseSample("OK", data, System.currentTimeMillis());
        ctx.channel().writeAndFlush(responseSample);
    }
}
```

```java
@Data
public class ResponseSample {
    private String code;
    private String data;
    private long timestamp;

    public ResponseSample(String code, String data, long timestamp) {
        this.code = code;
        this.data = data;
        this.timestamp = timestamp;
    }
}
```

#### 2. bind()

`bind() ` 才是真正进行服务器端口绑定和启动的入口，`sync()` 表示阻塞等待服务器启动完成。

我们来看下`bind()` 源码：

```java
// AbstractBootstrap
public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
    }

    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 初始化并注册 Channel，同时返回一个 ChannelFuture 实例 regFuture
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        // 判断 initAndRegister() 的过程是否发生异常，如果发生异常则直接返回。
        if (regFuture.cause() != null) {
            return regFuture;
        }
        // initAndRegister() 是否执行完毕，如果执行完毕则调用 doBind0() 进行 Socket 绑定。如果 initAndRegister() 还没有执行结束，regFuture 会添加一个 ChannelFutureListener 回调监听，当 initAndRegister() 执行结束后会调用 operationComplete()，同样通过 doBind0() 进行端口绑定。
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```

第一步validate()就不说了，不是重点，核心是 doBind():

在代码注释中也说的非常清楚，doBind()主要是两步：一是 `initAndRegister()`负责 **Channel 初始化注册**，返回的Future对象表明它是异步的，二是`doBind0()` **端口绑定**

#### 2.1 initAndRegister()

```java
// AbstractBootstrap
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();//创建 Channel
            init(channel);// 初始化 Channel
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);// 注册 Channel
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

注释也写了，主要分为三步

1. 创建Channel
2. 初始化Channel
3. 注册Channel

##### 2.1.1 创建Channel

channelFactory.newChannel()：
该实现类为 

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Constructor<? extends T> constructor;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            return constructor.newInstance();//反射创建
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(ReflectiveChannelFactory.class) +
                '(' + StringUtil.simpleClassName(constructor.getDeclaringClass()) + ".class)";
    }
}

```

可以看出，是通过反射创建了对象，那我们只需关注 constructor。
首先我们先关注 ReflectiveChannelFactory 是何时创建的？ 在EchoServer 中，我们通过`channel(NioServerSocketChannel.class)`创建出工厂。

```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

由此可见 constructor 是 NioServerSocketChannel 的构造方法

```java
    /**
     * Create a new instance
     */
    public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }

    /**
     * Create a new instance using the given {@link SelectorProvider}.
     */
    public NioServerSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }

    /**
     * Create a new instance using the given {@link ServerSocketChannel}.
     */
    public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
    
    private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
             *  {@link SelectorProvider#provider()} which is called by each ServerSocketChannel.open() otherwise.
             *
             *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
             */
            return provider.openServerSocketChannel();
        } catch (IOException e) {
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
    }
    
```

我们使用的是无参构造方法，不过他还是调用了`NioServerSocketChannel(ServerSocketChannel channel)`构造函数。

SelectorProvider 是nio中的一个类，通过 `openServerSocketChannel()` 可以创建服务端的 ServerSocketChannel 。而且 SelectorProvider 会根据操作系统类型和版本的不同，返回不同的实现类

```java
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```

然后构造函数通过super()调用了父类的构造器，我们debug进去可以定位到 AbstractNioChannel 和 AbstractChannel 中去。

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);// 调用 AbstractChannel 构造函数
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);//设置 Channel 是非阻塞模式
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            logger.warn(
                        "Failed to close a partially initialized socket.", e2);
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();// Channel 全局唯一 id 
    unsafe = newUnsafe();// unsafe 操作底层读写
    pipeline = newChannelPipeline();// pipeline 负责业务处理器编排
}
```

AbstractChannel 的构造函数创建了三个重要的成员变量,作用在注释中写了。

初始化状态，pipeline 的内部结构只包含头尾两个节点。三个核心成员变量创建好之后，会回到 AbstractNioChannel 的构造函数。

##### 2.1.2 初始化Channel

回到 ServerBootstrap 的 initAndRegister() 方法中的 `init()`

```java
void init(Channel channel) {
        setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);// 设置 Socket 参数
        setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));// 保存用户自定义属性

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions =
                childOptions.entrySet().toArray(newOptionArray(0));
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
        // 添加特殊的 Handler 处理器
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                //通过异步 task 的方式又向 Pipeline 添加一个处理器 ServerBootstrapAcceptor
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

`init()` 方法主要做了两件事

1. **设置Socket参数以及用户自定义属性**，在创建服务端 Channel 时，Channel 的配置参数保存在 NioServerSocketChannelConfig 中，在初始化 Channel 的过程中，**Netty 会将这些参数设置到 JDK 底层的 Socket 上，并把用户自定义的属性绑定在 Channel 上。**

2. **添加特殊的 Handler 处理器**。添加了一个 ChannelInitializer，ChannelInitializer 是实现了 ChannelHandler 接口的匿名类，`initChannel()` 方法用于添加 ServerSocketChannel 对应的 Handler。

   > ServerBootstrapAcceptor 是一个连接接入器，专门用于接收新的连接，然后把事件分发给 EventLoop 执行。

此时服务端的 pipeline 内部结构又发生了变化，如下图所示：

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20210223/Ciqc1F_V_y-Adn6LAAKfSPwUO3g505.png?versionId=CAEQKRiBgIDtp8i4vhciIDQxYWVhMDFiYTkzYzQyMTI4ZDc1YjM5ZTliNmNmODg3)

> 为何添加 ChannelInitializer 处理器？
>
> 我们在初始化时，还未将Channel注册到Selector上，也就无法注册Accept时间到Selector上，然后我们添加了 ChannelInitializer，里面是异步注册 ServerBootstrapAcceptor ，这样等待 Channel 注册完后，会向 Pipeline 中添加 ServerBootstrapAcceptor 处理器。

##### 2.1.3 注册Channel

register() 调用如下图

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20210223/QQ图片20210223153146.png?versionId=CAEQKRiBgID3usi4vhciIGZhZGQxZmM0MzI3YzQwN2ZhMDU3MDIxNzZlYWZjZmQw)

我们需要关注两个点：

```java
// MultithreadEventLoopGroup#register
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

`next()`是获得一个NioEventLoop，然后注册 Channel。
继续进入 register()

```java
// AbstractChannel#register
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    if (eventLoop == null) {
        throw new NullPointerException("eventLoop");
    }
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {// Reactor 线程内部调用
        register0(promise);
    } else {// 外部线程调用
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

Netty 会在线程池 EventLoopGroup 中选择一个 EventLoop 与当前 Channel 进行绑定，之后 Channel 生命周期内的所有 I/O 事件都由这个 EventLoop 负责处理，如 accept、connect、read、write 等 I/O 事件。不管是 EventLoop 线程本身调用，还是外部线程用，最终都会通过 `register0()` 方法进行注册：

```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister(); // 调用 JDK 底层的 register() 进行注册
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();// 触发 handlerAdded 事件

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();// 触发 channelRegistered 事件
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        // 此时 Channel 还未注册绑定地址，所以处于非活跃状态
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();// Channel 当前状态为活跃时，触发 channelActive 事件
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

**a.调用 JDK 底层的 register() 进行注册**  
**doRegister()**:

```java
#io.netty.channel.nio.AbstractNioChannel#doRegister
doRegister(){
    selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
    // 其余代码省略
}
```

```java
#java.nio.channels.spi.AbstractSelectableChannel#register
public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
            }
            return k;
        }
    }
```

register() 的第三个入参传入的是 Netty 自己实现的 Channel 对象，调用 register() 方法会将它绑定在 JDK 底层 Channel 的 attachment 上。这样在每次 Selector 对象进行事件循环时，Netty 都可以从返回的 JDK 底层 Channel 中获得自己的 Channel 对象。

**b.触发 handlerAdded 事件** 
`pipeline.invokeHandlerAddedIfNeeded()`会**把用户自定义的业务处理器添加到 Pipeline 中**

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20210223/QQ图片20210223165606.png?versionId=CAEQKRiBgIDYzMi4vhciIGE3YTgxNTZmODA2MzRlNDNhYjAyMzRkZDVkNjIyODQx)

这个方法里面调用还是很多的，有些复杂。
直接关注 `ChannelInitializer#initChannel()`

```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
        }
        return true;
    }
    return false;
}
```

继续深入 ininChannel() 方法，它的实现其实是 之前看到的 ServerBoorStrap中init()有定义匿名类：

```java
p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
```

这个里面之前也说过了：首先向 Pipeline 中添加 ServerSocketChannel 对应的 Handler，然后通过异步 task 的方式向 Pipeline 添加 ServerBootstrapAcceptor 处理器。**handler() 方法是添加到服务端的Pipeline** 上，而 **childHandler()方法是添加到客户端的Pipeline**上。所以对应 Echo 服务器示例中，此时被添加的是 LoggingHandler 处理器。

因为添加 ServerBootstrapAcceptor 是一个异步过程，需要 EventLoop 线程负责执行。而当前 EventLoop 线程正在执行 register0() 的注册流程，所以等到 register0() 执行完之后才能被添加到 Pipeline 当中。完成 initChannel() 这一步之后，ServerBootstrapAcceptor 并没有被添加到 Pipeline 中，此时 Pipeline 的内部结构变化如下图所示。

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20210223/Ciqc1F_V_5eAUGuoAAJsI9pISJU272.png?versionId=CAEQKRiBgIDwzoC5vhciIDJlMDM1ZDcwMjcwMjQ1MjI5Y2ZmNjE0OTE5NjQ4NGFk)

channelRegistered 事件是由 fireChannelRegistered() 方法触发，沿着 Pipeline 的 Head 节点传播到 Tail 节点，并依次调用每个 ChannelHandler 的 channelRegistered() 方法。然而此时 Channel 还未注册绑定地址，所以处于非活跃状态，所以并不会触发 channelActive 事件。

执行完整个 register0() 的注册流程之后，EventLoop 线程会将 ServerBootstrapAcceptor 添加到 Pipeline 当中，此时 Pipeline 的内部结构又发生了变化，如下图所示。

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20210223/Ciqc1F_V_7SAUkiOAAI_yPZd0NI396.png?versionId=CAEQKRiBgICX44C5vhciIGY1MmYzMjYxYTMyODRiNDA4ZjE1NDlhODAxNWVkNGJm)

至此，整个服务端 Channel 注册的流程才结束，然后就是端口绑定。

#### doBind()

在`initAndRegister()` 之后调用`dobind0()`

```java
private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

 这个调用也挺深，我就直接debug进入

![image](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/QQ图片20210224133524.png)

```java
// NioServerSocketChannel
@Override
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
       	// 我是用的jdk8，这里就是调用jdk底层的bind()方法了。具体就不进去看了
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

在bind完成后会调用 pipeline.fireChannelActive() 方法触发 channelActive 事件：

```java
// io.netty.channel.AbstractChannel.AbstractUnsafe#bind
if (!wasActive && isActive()) {
    invokeLater(new Runnable() {
        @Override
        public void run() {
            pipeline.fireChannelActive();
        }
    });
}
```

继续debug,在执行完 channelActive 事件传播之后，会调用 readIfIsAutoRead() 方法触发 Channel 的 read 事件：

```java
// io.netty.channel.DefaultChannelPipeline.HeadContext#channelActive
@Override
public void channelActive(ChannelHandlerContext ctx) {
    ctx.fireChannelActive();

    readIfIsAutoRead();
}
```

![image-20210226132352797](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/image-20210226132352797.png)

```java
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);// 注册 OP_ACCEPT 事件到服务端 Channel 的事件集合
    }
}
```

这里 readInterestOp = 16。实际上他是 AbstractNioChannel 的一个属性，而且赋值的时机是之前说的 创建 Channel 时：

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

SelectionKey.OP_ACCEPT 值为16。所以 OP_ACCEPT 事件会被注册到 Channel 的事件集合中。

到此为止，整个服务端已经真正启动完毕。总结来说就是4步：

1. **创建服务端 Channel** ：本质是创建 JDK 底层原生的 Channel，并初始化几个重要的属性，包括 id、unsafe、pipeline 等。
2. **初始化服务端 Channel** ：设置 Socket 参数以及用户自定义属性，并添加两个特殊的处理器 ChannelInitializer 和 ServerBootstrapAcceptor。
3. **注册服务端 Channel**：调用JDK底层将 Channel 注册到 Selector 上。
4. **端口绑定**：调用 JDK 底层进行端口绑定，并触发 channelActive 事件，把 OP_ACCEPT 事件注册到 Channel 的事件集合中。

调用链路图:

![CgqCHl_WABCAJA9VAAIG0Ncq3hs061](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgqCHl_WABCAJA9VAAIG0Ncq3hs061.png)

> 参考
>
> [Netty 核心原理剖析与 RPC 实践 ](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=516#/detail/pc?id=4931)

---

### Netty源码（二）：服务端如何处理客户端新建连接

Netty 服务端是如何处理客户端新建连接的呢？主要分为四步：

1. Boos NioEventLoop 线程轮询客户端连接 OP_ACCEPT事件；
2. 构造 Netty 客户端 NioSocketChannel；
3. 注册 Netty 客户端 NioSocketChannel 到 Worker 工作线程中；
4. 注册 OP_HEAD 事件到 NioSocketChannel 的事件集合。

#### 1.Netty 中 Boss NioEventLoop 专门负责接收新的连接

NioEventLoop 下次再说，当客户端有新连接接入服务端时，Boss NioEventLoop 会监听到 OP_ACCEPT 事件，源码如下所示：

```java
// NioEventLoop#processSelectedKey
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}

```

```java
// NioMessageUnsafe.read() 
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle(); 
    allocHandle.reset(config);
    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                int localRead = doReadMessages(readBuf);  // while 循环不断读取 Buffer 中的数据
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }
                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i)); // 传播读取事件
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete(); // 传播读取完毕事件
        // 省略其他代码
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}

```

```java
// NioServerSocketChannel#doReadMessages()
protected int doReadMessages(List<Object> buf) throws Exception {
    // Netty 先通过 JDK 底层的 accept() 获取 JDK 原生的 SocketChannel
    SocketChannel ch = SocketUtils.accept(javaChannel());
    try {
        if (ch != null) {
            // 封装成 Netty 自己的 NioSocketChannel
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }
    return 0;
}

```

#### 2.构造 Netty 客户端 NioSocketChannel

上面的代码其实就是 new NioSocketChannel(this, ch) - 构造 Netty 客户端 NioSocketChannel。新建 Netty 的客户端 Channel 的实现原理与上文中我们讲到的创建服务端 Channel 的过程是类似的，只是服务端 Channel  的类型是 NioServerSocketChannel，而**客户端 Channel 的类型是  NioSocketChannel**。NioSocketChannel 的创建同样会完成几件事：创建核心成员变量  id、unsafe、pipeline；注册 SelectionKey.OP_READ 事件；设置 Channel 的为非阻塞模式；新建客户端  Channel 的配置。

对于服务端来说，此时 Pipeline 的内部结构如下图所示。

![CgqCHl_V__eAOXbQAALQ0Dt6N64798](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgqCHl_V__eAOXbQAALQ0Dt6N64798.png)

ServerBootstrapAcceptor 是一个连接接入器，专门用于接收新的连接，然后把事件分发给 EventLoop 执行,它是在上篇中初始化Channel的时候创建的。

成功构造客户端 NioSocketChannel 后，接下来会通过 pipeline.fireChannelRead() 触发 channelRead 事件传播，channelRead 事件会传播到 ServerBootstrapAcceptor.channelRead() 方法：将客户端 Channel 分配到工作线程组中去执行。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 强制转换为 Channel
    // ServerBootstrapAcceptor 是服务端 Channel 中一个特殊的处理器，而服务端 Channel 的 channelRead 事件只会在新连接接入时触发，所以这里拿到的数据msg都是客户端新连接。
    final Channel child = (Channel) msg;
    // 在客户端 Channel 中添加 childHandler，childHandler 是用户在启动类中通过 childHandler() 方法指定的
    child.pipeline().addLast(childHandler);
    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);
    try {
        // 注册客户端 Channel
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

```

#### 3.注册 Netty 客户端 NioSocketChannel 到 Worker 工作线程中



#### 4.注册 OP_HEAD 事件到 NioSocketChannel 的事件集合



这两步都是上面的 childGroup.register() 完成的。具体暂时不清楚，需要debug源码才知道。

---

### Netty源码（三） Reactor 线程模型        

上次说道 NioEventLoop 之后再说，现在就来说下这个。

在 Netty 中 EventLoop 是 Reactor 线程模型的核心处理引擎，Reactor 线程模型是 Netty 实现高性能的核心所在。因为Netty是基于Nio实现的，推荐使用的是NioEventLoop，其核心入口为run():

```java
protected void run() {
        for (;;) {
            try {
                try {
                    switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));// 1.轮询 I/O事件

                        // 'wakenUp.compareAndSet(false, true)' is always evaluated
                        // before calling 'selector.wakeup()' to reduce the wake-up
                        // overhead. (Selector.wakeup() is an expensive operation.)
                        //
                        // However, there is a race condition in this approach.
                        // The race condition is triggered when 'wakenUp' is set to
                        // true too early.
                        //
                        // 'wakenUp' is set to true too early if:
                        // 1) Selector is waken up between 'wakenUp.set(false)' and
                        //    'selector.select(...)'. (BAD)
                        // 2) Selector is waken up between 'selector.select(...)' and
                        //    'if (wakenUp.get()) { ... }'. (OK)
                        //
                        // In the first case, 'wakenUp' is set to true and the
                        // following 'selector.select(...)' will wake up immediately.
                        // Until 'wakenUp' is set to false again in the next round,
                        // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                        // any attempt to wake up the Selector will fail, too, causing
                        // the following 'selector.select(...)' call to block
                        // unnecessarily.
                        //
                        // To fix this problem, we wake up the selector again if wakenUp
                        // is true immediately after selector.select(...).
                        // It is inefficient in that it wakes up the selector for both
                        // the first case (BAD - wake-up required) and the second case
                        // (OK - no wake-up required).

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                    // the selector and retry. https://github.com/netty/netty/issues/8566
                    rebuildSelector0();
                    handleLoopException(e);
                    continue;
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();// 2 处理I/O事件
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();// 3 处理所有任务
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();// 2 处理I/O事件
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);// 3 处理完 I/O 事件，再处理异步任务队列
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

run() 方法是一个无限循环，没有任何退出条件，在不间断循环执行以下三件事情，可以用下面这张图形象地表示:

![Cip5yF_ZyhCAM3GUAASrdEcuR2U593](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Cip5yF_ZyhCAM3GUAASrdEcuR2U593.png)

1. **轮询 I/O 事件**（select）：轮询 Selector 选择器中已经注册的所有 Channel 的 I/O 事件。
2. **处理 I/O 事件**（processSelectedKeys）：处理已经准备就绪的 I/O 事件。
3. **处理异步任务队列**（runAllTasks）：Reactor 线程还有一个非常重要的职责，就是处理任务队列中的非 I/O 任务。Netty 提供了 ioRatio 参数用于调整 I/O 事件处理和任务处理的时间比例。

#### 1.Select

首先看第一步，轮询。第一步switch决定选择不同的策略。生成策略的逻辑是`selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())`

```java
// io.netty.channel.DefaultSelectStrategy#calculateStrategy
public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
    // 当前 NioEventLoop 线程存在异步任务， 最终调用到 selectNow() 方法，selectNow() 是非阻塞，执行后立即返回
    // NioEventLoop 线程的不存在异步任务，即任务队列为空，返回的是 SELECT 策略
    return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
}
// NioEventLoop#selectNowSupplier
private final IntSupplier selectNowSupplier = new IntSupplier() {
    @Override
    public int get() throws Exception {
        return selectNow();
    }
};
// NioEventLoop#selectNow
int selectNow() throws IOException {
    try {
        return selector.selectNow();
    } finally {
        // restore wakeup state if needed
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}
```

选择策略后进入switch判断：首先上面获得策略中，如果当期那线程存在异步任务，switch会进入default分支直接跳出。

```java
case SelectStrategy.CONTINUE:
    continue;//下一次循环
case SelectStrategy.BUSY_WAIT:// fall through 啥也不干，执行下面逻辑
case SelectStrategy.SELECT:
    select(wakenUp.getAndSet(false));
    if (wakenUp.get()) {
        selector.wakeup();
    }
default:
```

上面的策略选择其实就是确保如果有存在异步任务，NioEventLoop 会优先保证 CPU 能够及时处理异步任务。

当 NioEventLoop 线程的不存在异步任务，即任务队列为空，返回的是 SELECT 策略, 就会调用 select(boolean oldWakenUp) 方法：

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        // 用于记录 select 操作的次数
        int selectCnt = 0;
        // 当前时间
        long currentTimeNanos = System.nanoTime();
        // selectDeadLineNanos：定时任务队列中最近待执行任务的执行时间
        // Netty 中定时任务队列是按照延迟时间从小到大进行排列的，
        // delayNanos(currentTimeNanos)获得第一个待执行定时任务的延迟时间
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        long normalizedDeadlineNanos = selectDeadLineNanos - initialNanoTime();
        if (nextWakeupTime != normalizedDeadlineNanos) {
            nextWakeupTime = normalizedDeadlineNanos;
        }

        for (;;) {
            // 1.检测 select 阻塞操作是否超过截止时间
            // 判断 currentTimeNanos 是否超过 selectDeadLineNanos 0.5ms 以上
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) {
                // 如果超过说明当前任务队列中有定时任务需要立刻执行
                if (selectCnt == 0) {
                    // 退出之前如果从未执行过 select 操作，那么会立即一次非阻塞的 selectNow 操作
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            // 2. 轮询过程中如果有任务产生，中断本次轮询
            // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
            // Selector#wakeup. So we need to check task queue again before executing select operation.
            // If we don't, the task might be pended until select operation was timed out.
            // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;//立即跳出当前循环回到上层循环的主流程，确保接下来能够优先执行 runAllTasks
            }
            // 3. select 阻塞等待获取 I/O 事件
            // 阻塞等待 timeoutMillis 的超时时间。如果定时任务的截止时间非常久,可能一直阻塞Netty无法工作，所以 Netty 在外部线程添加任务的时候，可以唤醒 select 阻塞操作。
            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                // - Selected something,
                // - waken up by user, or
                // - the task queue has a pending task.
                // - a scheduled task is ready for processing
                break;
            }
            if (Thread.interrupted()) {
                // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                // As this is most likely a bug in the handler of the user or it's client library we will
                // also log it.
                //
                // See https://github.com/netty/netty/issues/2426
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }
            // 4. 解决臭名昭著的 JDK epoll 空轮询 Bug
            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // 如果事件轮询时间小于 timeoutMillis，并且在该时间周期内连续发生超过 SELECTOR_AUTO_REBUILD_THRESHOLD（默认512） 次空轮询，说明可能触发了 epoll 空轮询 Bug, Netty会重建新的 Selector 对象，将异常的 Selector 中所有的 SelectionKey 会重新注册到新建的 Selector
                // The code exists in an extra method to ensure the method is not too big to inline as this
                // branch is not very likely to get hit very frequently.
                selector = selectRebuildSelector(selectCnt);
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                        selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                    selector, e);
        }
        // Harmless exception - log anyway
    }
}
```

- Netty 的任务队列包括普通任务、定时任务以及尾部任务，hasTask() 判断的是普通任务队列和尾部队列是否为空，而 delayNanos(currentTimeNanos) 方法获取的是定时任务的延迟时间。

- 第三步中说道：Netty 在外部线程添加任务的时候，可以唤醒 select 阻塞操作。

  ```java
  // selector.wakeup() 操作的开销非常大,在每次调用之前都会先执行 wakenUp.compareAndSet(false, true)，只有设置成功之后才会执行 selector.wakeup() 操作
  // SingleThreadEventExecutor#execute
  public void execute(Runnable task) {
    	  // 省略其他代码
      if (!addTaskWakesUp && wakesUpForTask(task)) {
          wakeup(inEventLoop); 
      }
  }
  
  // NioEventLoop#wakeup
  protected void wakeup(boolean inEventLoop) {
      // 如果是外部线程，设置 wakenUp 为true，则唤醒 select 阻塞操作
      if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
          selector.wakeup(); 
      }
  }
  ```

> 小结：select 操作也是一个无限循环，在事件轮询之前检查任务队列是否为空，确保任务队列中待执行的任务能够及时执行。如果任务队列中已经为空，然后执行  select 阻塞操作获取等待获取 I/O 事件。Netty 通过引入计数器变量，并统计在一定时间窗口内 select  操作的执行次数，识别出可能存在异常的 Selector 对象，然后采用重建 Selector 的方式巧妙地避免了 JDK epoll  空轮询的问题。

2. #### 处理I/O事件

回到 run() 方法第二步，调用 processSelectedKeys() 方法处理I/O事件。

```java
// run()
final int ioRatio = this.ioRatio;
if (ioRatio == 100) {
    try {
        processSelectedKeys();// 2 处理I/O事件
    } finally {
        // Ensure we always run tasks.
        runAllTasks();// 3 处理所有任务
    }
} else {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys();// 2 处理I/O事件
    } finally {
        // Ensure we always run tasks.
        final long ioTime = System.nanoTime() - ioStartTime;
        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);// 3 处理完 I/O 事件，再处理异步任务队列
    }
}
```

默认为 ioRatio = 50。如果 ioRatio = 100,表示每次都处理完 I/O 事件后，会执行所有的 task。如果 ioRatio < 100，也会优先处理完 I/O 事件，再处理异步任务队列。所以不论如何 processSelectedKeys() 都是先执行的：

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();// Netty 优化逻辑
    } else {
        processSelectedKeysPlain(selector.selectedKeys());// 正常逻辑
    }
}
```

如果 selectedKeys 不为空，则去处理 Netty 优化过的集合，否则走正常处理。selectedKeys  是 Netty 优化过的集合，类型为 SelectedSelectionKeySet ，正常逻辑使用的是 JDK 的 HashSet 类型。

##### 2.1 processSelectedKeysPlain

```java
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
    // check if the set is empty and if so just return to not create garbage by
    // creating a new Iterator every time even if there is nothing to process.
    // See https://github.com/netty/netty/issues/597
    if (selectedKeys.isEmpty()) {
        return;
    }

    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        // 遍历依次处理已经就绪的 SelectionKey，拿到上面挂载的 attachment
        final SelectionKey k = i.next();
        final Object a = k.attachment();
        i.remove();
		// SelectionKey 的类型可能是 AbstractNioChannel 和 NioTask
        if (a instanceof AbstractNioChannel) {
            // I/O 事件由 Netty 处理
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            // NioTask 是用户自定义的 task，一般不会是这种类型
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (!i.hasNext()) {
            break;
        }

        if (needsToSelectAgain) {
            selectAgain();
            selectedKeys = selector.selectedKeys();

            // Create the iterator again to avoid ConcurrentModificationException
            if (selectedKeys.isEmpty()) {
                break;
            } else {
                i = selectedKeys.iterator();
            }
        }
    }
}
```

processSelectedKey() 很明显是两个重构方法：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {// 检查 Key 是否合法
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;// 这里是要关闭ch没，如果没有注册EventLoop，那就直接return。
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());// Key 不合法，直接关闭连接
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // 处理连接事件
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }
		// 处理可写事件
        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }
		// 处理可读事件
        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

- **OP_CONNECT** **连接建立事件**。表示 TCP 连接建立成功, Channel 处于 Active 状态。处理 OP_CONNECT 事件**首先将该事件从事件集合中清除**，避免事件集合中一直存在连接建立事件，然后调用 unsafe.finishConnect() 方法通知上层连接已经建立。可以跟进 unsafe.finishConnect() 的源码发现会底层调用的 pipeline().fireChannelActive() 方法，这时会产生一个 Inbound 事件，然后会在 Pipeline 中进行传播，依次调用 ChannelHandler 的 channelActive() 方法，通知各个 ChannelHandler 连接建立成功。
- **OP_WRITE，可写事件**。表示上层可以向 Channel 写入数据，通过执行 ch.unsafe().forceFlush() 操作，将数据冲刷到客户端，最终会调用 javaChannel 的 write() 方法执行底层写操作。
- **OP_READ，可读事件**。表示 Channel 收到了可以被读取的新数据。Netty 将 **READ 和 Accept 事件进行了统一的封装**，都通过 unsafe.read() 进行处理。unsafe.read() 的逻辑可以归纳为几个步骤：从 Channel 中读取数据并存储到分配的 ByteBuf；调用 pipeline.fireChannelRead() 方法产生 Inbound 事件，然后依次调用 ChannelHandler 的 channelRead() 方法处理数据；调用 pipeline.fireChannelReadComplete() 方法完成读操作；最终执行 removeReadOp() 清除 OP_READ 事件。

我们再次回到 processSelectedKeysPlain 的主流程，接下来会判断 needsToSelectAgain 决定是否需要重新轮询。如果 needsToSelectAgain == true，会调用 selectAgain() 方法进行重新轮询，该方法会将 needsToSelectAgain 再次置为 false，然后调用 selectorNow() 后立即返回。

> **什么场景下 needsToSelectAgain 会再次设置为 true**?
>
> 当 Netty 在处理 I/O 事件的过程中，如果发现超过默认阈值 256 个 Channel 从 Selector 对象中移除后，会将 needsToSelectAgai 设置为 true，重新做一次轮询操作，从而确保 keySet 的有效性。
>
> ```java
> // AbstractChannel#doDeregister
> protected void doDeregister() throws Exception {
>     eventLoop().cancel(selectionKey());
> }
> void cancel(SelectionKey key) {
>     key.cancel();
>     cancelledKeys ++;
>     // 当取消的 Key 超过默认阈值 256，needsToSelectAgain 设置为 true
>     if (cancelledKeys >= CLEANUP_INTERVAL) {
>         cancelledKeys = 0;
>         needsToSelectAgain = true;
>     }
> }
> ```
>
> 

##### 2.2 processSelectedKeysOptimized

再来分析 Netty 优化的 processSelectedKeysOptimized() :

```java
private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;

            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }

            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);

                selectAgain();
                i = -1;
            }
        }
    }
```

其代码与 processSelectedKeysPlain 颇为相似。不同之处在于 selectedKeys 的遍历方式。故需要知悉 SelectedSelectionKeySet 内部结构：

```java
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {
	// 可以看到其内部使用的数组进行封装，相比 JDK HashSet 的遍历效率更高。
    SelectionKey[] keys;
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        keys[size++] = o;
        if (size == keys.length) {
            increaseCapacity();
        }

        return true;
    }
    // 其余略
}
```

> SelectedSelectionKeySet 内部通过 size 变量记录数据的逻辑长度，每次执行 add 操作时，会把对象添加到 SelectionKey[] 尾部。当 size 等于 SelectionKey[] 的真实长度时，SelectionKey[] 会进行扩容。相比于 HashSet，SelectionKey[] 不需要考虑哈希冲突的问题，所以可以实现 O(1) 时间复杂度的 add 操作。

我们还需关注 SelectedSelectionKeySet  是何处生成的，在 NioEventLoop 中，根据其引用名 selectedKeys 找到赋值位置为 NioEventLoop#openSelector 方法中：

```java
private SelectorTuple openSelector() {

		// 省略其余代码
        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                        // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                        // This allows us to also do this in Java9+ without any extra flags.
                        long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset =
                                PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                        if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                    unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                            PlatformDependent.putObject(
                                    unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection as last-resort.
                    }
                    Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
        });
        selectedKeys = selectedKeySet;
    }
```

可以看出，Netty 通过反射的方式，将 Selector 对象内部的 selectedKeys 和 publicSelectedKeys 替换为 SelectedSelectionKeySet，原先 selectedKeys 和 publicSelectedKeys 这两个字段都是 HashSet 类型。

> 小结：Reactor 线程主流程的第二步，处理 I/O 事件 processSelectedKeys 已经说完。处理 I/O 事件时有两种选择，一种是处理 Netty 优化过的 selectedKeys，另外一种是正常的处理逻辑，两种策略的处理逻辑是相似的，都是通过获取 SelectionKey 上挂载的 attachment 判断 SelectionKey 的类型，不同的 SelectionKey 的类型又会调用不同的处理方法，然后通过 Pipeline 进行事件传播。Netty 优化过的 selectedKeys 是使用数组存储的 SelectionKey，相比于 JDK 的 HashSet 遍历效率更高效。processSelectedKeys 还做了更多的优化处理，如果发现超过默认阈值 256 个 Channel 从 Selector 对象中移除后，会重新做一次轮询操作，以确保 keySet 的有效性。

#### 3 处理异步任务队列

继续回到 run() 方法，来到第三步：处理异步任务队列。

##### 3.1 任务添加

NioEventLoop 内部有两个非常重要的异步任务队列，分别为**普通任务队列和定时任务队列**。NioEventLoop 提供了 execute() 和 schedule() 方法用于向不同的队列中添加任务，execute() 用于添加普通任务，schedule() 方法用于添加定时任务。

###### 3.1.2 添加普通任务

NioEventLoop 继承自 SingleThreadEventExecutor，SingleThreadEventExecutor 提供了 execute() 用于添加普通任务：

```java
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);// 添加任务
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}

protected void addTask(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (!offerTask(task)) {
        reject(task);
    }
}

final boolean offerTask(Runnable task) {
    if (isShutdown()) {
        reject();
    }
    return taskQueue.offer(task);
}
```

可见他把人物放进了 taskQueue 队列之中，`taskQueue = newTaskQueue(this.maxPendingTasks);`在 NioEventLoop 中的实现为：

```java
@Override
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    return newTaskQueue0(maxPendingTasks);
}

private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
    // This event loop never calls takeTask()
    return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()
        : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
}
```

可见 taskQueue 使用的是 Mpsc Queue(无锁队列)，可以理解为多生产者单消费者队列，以后再详解。

> 在任务处理的场景下，inEventLoop() 始终是返回 true，始终都是在 Reactor 线程内执行，既然在 Reactor 线程内都是串行执行，可以保证线程安全，那为什么还需要 Mpsc Queue ？答案是**为了解决外部线程添加定时任务的线程安全问题**。

**外部线程调用 Channel 的相关方法**, 比如在 RPC 业务线程池里处理完业务请求后，可以根据用户请求拿到关联的 Channel，将数据写回客户端,channel.write() :

```java
// #AbstractChannel#write
public ChannelFuture write(Object msg) {
    return pipeline.write(msg);
}
// AbstarctChannelHandlerContext#write
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ObjectUtil.checkNotNull(msg, "msg");
    try {
        if (isNotValidPromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }

    final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                                                                   (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    // 如果是 Reactor 线程发起调用 channel.write() 方法，inEventLoop() 返回 true
    if (executor.inEventLoop()) {// Reactor 线程内部调用
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {// 外部线程调用
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}
// AbstractChannelHandlerContext#safeExecute
private static boolean safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg) {
    try {
        executor.execute(runnable);
        return true;
    } catch (Throwable cause) {
        try {
            promise.setFailure(cause);
        } finally {
            if (msg != null) {
                ReferenceCountUtil.release(msg);
            }
        }
        return false;
    }
}
```

外部线程调用时会将写操作封装成一个 WriteTask，然后通过 safeExecute() 执行，可以发现 safeExecute() 就是调用的  SingleThreadEventExecutor#execute() 方法，最终会将任务添加到 taskQueue  中。因为多个外部线程可能会并发操作同一个 Channel，这时候 Mpsc Queue 就可以保证线程的安全性。

###### 3.1.3 添加定时任务

普通任务类似，定时任务也会有 Reactor 线程内和外部线程两种场景，我们直接跟进到 AbstractScheduledEventExecutor#schedule() ：

```java
private <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {// Reactor 线程内部
        scheduledTaskQueue().add(task.setId(nextTaskId++));
    } else {// 外部线程
        executeScheduledRunnable(new Runnable() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task.setId(nextTaskId++));
            }
        }, true, task.deadlineNanos());
    }

    return task;
}
PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue() {
    if (scheduledTaskQueue == null) {
        // scheduledTaskQueue 的默认实现是优先级队列 DefaultPriorityQueue
        scheduledTaskQueue = new DefaultPriorityQueue<ScheduledFutureTask<?>>(
            SCHEDULED_FUTURE_TASK_COMPARATOR,
            // Use same initial capacity as java.util.PriorityQueue
            11);
    }
    return scheduledTaskQueue;
}
void executeScheduledRunnable(Runnable runnable,
                              @SuppressWarnings("unused") boolean isAddition,
                              @SuppressWarnings("unused") long deadlineNanos) {
    execute(runnable);// 普通任务的 execute() 
}
```

DefaultPriorityQueue 是非线程安全的，如果是 Reactor  线程内部调用，因为是串行执行，所以不会有线程安全问题。如果是外部线程添加定时任务，我们发现 Netty  把添加定时任务的操作又再次封装成一个任务交由 executeScheduledRunnable() 处理，而  executeScheduledRunnable() 中又**再次调用了普通任务的 execute() 的方法**，巧妙地**借助普通任务场景中 Mpsc  Queue 解决了外部线程添加定时任务的线程安全问题**。

##### 3.2 任务执行

处理异步任务队列有 runAllTasks() 和 runAllTasks(long timeoutNanos)  两种实现，第一种会处理所有任务，第二种是带有超时时间来处理任务。之所以设置超时时间是为了防止 Reactor 线程处理任务时间过长而导致 I/O 事件阻塞，我们着重分析下 runAllTasks(long timeoutNanos) 

```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();// 1. 合并定时任务到普通任务队列
    Runnable task = pollTask();// 2. 从普通任务队列中取出任务并处理
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }
	// 计算任务处理的超时时间
    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);// 执行任务

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive. 每执行 64 个任务检查一下是否超时
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();// 继续取出下一个任务
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();// 3. 收尾工作
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```

异步任务处理 runAllTasks 的过程可以分为三步：**合并定时任务到普通任务队列，然后从普通任务队列中取出任务并处理，最后进行收尾工作。**

###### 3.2.1 合并定时任务到普通任务队列

对应的实现是 fetchFromScheduledTaskQueue() ：

```java
private boolean fetchFromScheduledTaskQueue() {
    if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty()) {
        return true;
    }
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    for (;;) {
        // 从定时任务队列中取出截止时间小于等于当前时间的定时任务
        Runnable scheduledTask = pollScheduledTask(nanoTime);
        if (scheduledTask == null) {
            return true;
        }
        if (!taskQueue.offer(scheduledTask)) {// 普通任务队列已满
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue.add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
    }
}
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
     // 如果定时任务的 deadlineNanos 小于当前时间就取出
    if (scheduledTask == null || scheduledTask.deadlineNanos() - nanoTime > 0) {
        // 如果定时任务截止时间 deadlineNanos 小于当前时间，不满足条件，
        // 由于定时任务是按照截止时间 deadlineNanos 从小到大排列的说明后面的更不会满足，直接return null
        return null;
    }
    scheduledTaskQueue.remove();
    return scheduledTask;
}
```

###### 3.2.2 从普通任务队列中取出任务并处理

上面一段中已有注释，真正处理任务的 safeExecute() 就是直接调用 Runnable 的 run() 方法。因为异步任务处理是有超时时间的，所以 Netty 采取了定时检测的策略，每执行 64 个任务的时候就会检查一下是否超时，这也是出于对性能的折中考虑，**如果异步队列中有大量的短时间任务，每一次执行完都检测一次超时性能会有所降低**。

###### 3.2.3 收尾工作

afterRunningAllTasks()：

```java
protected void afterRunningAllTasks() {
    runAllTasksFrom(tailTasks);
}
protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
    Runnable task = pollTaskFrom(taskQueue);
    if (task == null) {
        return false;
    }
    for (;;) {
        safeExecute(task);
        task = pollTaskFrom(taskQueue);
        if (task == null) {
            return true;
        }
    }
}
```

tailTasks (尾部队列) 相比于普通任务队列优先级较低，可以理解为是收尾任务，在每次执行完 taskQueue  中任务后会去获取尾部队列中任务执行。可以看出 afterRunningAllTasks() 就是把尾部队列 tailTasks  里的任务以此取出执行一遍。尾部队列并不常用，一般用于对 Netty  的运行状态做一些统计数据，例如任务循环的耗时、占用物理内存的大小等等，都可以向尾部队列添加一个收尾任务完成统计数据的实时更新。

> 小结：异步任务主要分为普通任务和定时任务两种，在任务添加和任务执行时，都需要考虑 Reactor  线程内和外部线程两种情况。外部线程添加定时任务时，Netty 巧妙地借助普通任务的 Mpsc Queue  解决多线程并发操作时的线程安全问题。Netty 执行任务之前会将满足条件的定时任务合并到普通任务队列，由普通任务队列统一负责执行，并且每执行  64 个任务的时候就会检查一下是否超时。

#### 4 总结

NioEventLoop 的无限循环中所做的三件事：轮询 I/O 事件，处理 I/O 事件，处理异步任务队列。 

---

### Netty源码（四）网络请求处理
too much

---

### Netty源码（五）FastThreadLocal

FastThreadLocal 的实现与 ThreadLocal 非常类似，对于 ThreadLocal 不清楚的参考[[10. ThreadLocal](https://sunsas.gitee.io/doc/#/sunsas/Java多线程?id=_10-threadlocal)]。Netty 为 FastThreadLocal 量身打造了  **FastThreadLocalThread** 和 **InternalThreadLocalMap** 两个重要的类。下面我们看下这两个类是如何实现的。

FastThreadLocalThread 是对 Thread 类的一层包装，每个线程对应一个 InternalThreadLocalMap 实例：

```java
public class FastThreadLocalThread extends Thread {
    private InternalThreadLocalMap threadLocalMap;
    // 省略其他代码
}
```

threadLocalMap 替代 Thread 中的 ThreadLocalMap，所以得先了解 InternalThreadLocalMap ：

```java
public static final Object UNSET = new Object();

private InternalThreadLocalMap() {
    super(newIndexedVariableTable());
}

private static Object[] newIndexedVariableTable() {
    Object[] array = new Object[32];
    Arrays.fill(array, UNSET);
    return array;
}
```

从 InternalThreadLocalMap 内部实现来看，与 ThreadLocalMap 一样都是采用数组的存储方式。但是  InternalThreadLocalMap 并没有使用线性探测法来解决 Hash 冲突，而是在 FastThreadLocal  初始化的时候分配一个数组索引 index，index 的值采用原子类 AtomicInteger 保证顺序递增，通过调用  InternalThreadLocalMap.nextVariableIndex() 方法获得。

```java
// FastThreadLocal
private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();
// InternalThreadLocalMap
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement();
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
//  InternalThreadLocalMap 父类 UnpaddedInternalThreadLocalMap
static final AtomicInteger nextIndex = new AtomicInteger();
```

在读写数据的时候通过数组下标 index 直接定位到 FastThreadLocal 的位置，时间复杂度为 O(1)。如果数组下标递增到非常大，那么数组也会比较大，所以 FastThreadLocal 是通过空间换时间的思想提升读写性能。

```java
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }

    return initialize(threadLocalMap);
}
// InternalThreadLocalMap
public Object indexedVariable(int index) {
    Object[] lookup = indexedVariables;
    return index < lookup.length? lookup[index] : UNSET;
}
```

![Ciqc1F_qw1KAUXO0AAMZJ_Hk4dQ099](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Ciqc1F_qw1KAUXO0AAMZJ_Hk4dQ099.png)

FastThreadLocal 使用 Object 数组替代了 Entry 数组，**Object[0]**  存储的是一个Set<FastThreadLocal<?>> 集合，从**数组下标 1 开始**都是直接存储的 value  数据，不再采用 ThreadLocal 的键值对形式进行存储。

FastThreadLocal 的使用方法几乎和 ThreadLocal 保持一致,ThreadLocal 怎么用，FastThreadLocal 就怎么用。

FastThreadLocal.set() ：

```java
public final void set(V value) {
    if (value != InternalThreadLocalMap.UNSET) { // 1. value 是否为缺省值
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get(); // 2. 获取当前线程的 InternalThreadLocalMap
        setKnownNotUnset(threadLocalMap, value); // 3. 将 InternalThreadLocalMap 中数据替换为新的 value
    } else {
        remove();
    }
}
```

```java
// InternalThreadLocalMap.get()
public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        // 如果当前线程是 FastThreadLocalThread 类型，那么直接通过 fastGet() 方法获取 FastThreadLocalThread 的 threadLocalMap 属性即可
        return fastGet((FastThreadLocalThread) thread);
    } else {
        // 如果当前线程不是 FastThreadLocalThread,兜底
        return slowGet();
    }
}
private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null) {
        // 如果此时 InternalThreadLocalMap 不存在，直接创建一个返回
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}

private static InternalThreadLocalMap slowGet() {
    // UnpaddedInternalThreadLocalMap 中保存了一个 JDK 原生的 ThreadLocal，ThreadLocal 中存放着 InternalThreadLocalMap
    ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    // ThreadLocal 中存放着 InternalThreadLocalMap，此时获取 InternalThreadLocalMap 就退化成 JDK 原生的 ThreadLocal 获取
    InternalThreadLocalMap ret = slowThreadLocalMap.get();
    if (ret == null) {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret);
    }
    return ret;
}
```

回到 FastThreadLocal.set() 的第三步：setKnownNotUnset(threadLocalMap, value)：

```java
private void setKnownNotUnset(InternalThreadLocalMap threadLocalMap, V value) {
    // 1. 找到数组下标 index 位置，设置新的 value
    if (threadLocalMap.setIndexedVariable(index, value)) {
        // 2. 将 FastThreadLocal 对象保存到待清理的 Set 中
        addToVariablesToRemove(threadLocalMap, this);
    }
}
```

threadLocalMap.setIndexedVariable()：

```java
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value;// 直接将数组 index 位置设置为 value，时间复杂度为 O(1)
        return oldValue == UNSET;
    } else {
        expandIndexedVariableTableAndSet(index, value);// 容量不够，先扩容再设置值
        return true;
    }
}
// 扩容方法
private void expandIndexedVariableTableAndSet(int index, Object value) {
    Object[] oldArray = indexedVariables;
    final int oldCapacity = oldArray.length;
    int newCapacity = index;
    newCapacity |= newCapacity >>>  1;
    newCapacity |= newCapacity >>>  2;
    newCapacity |= newCapacity >>>  4;
    newCapacity |= newCapacity >>>  8;
    newCapacity |= newCapacity >>> 16;
    newCapacity ++;

    Object[] newArray = Arrays.copyOf(oldArray, newCapacity);
    Arrays.fill(newArray, oldCapacity, newArray.length, UNSET);
    newArray[index] = value;
    indexedVariables = newArray;
}
```

上面一段位移代码十分熟悉，之前在HashMap中详细介绍过此方法 tableSizeFor()，是为了取得某数向上最小的2次幂，确定容量。比如initialCapacity为30，则返回32。可参考[HashMap中的tableSizeFor方法](https://blog.csdn.net/Z_ChenChen/article/details/82955221)

扩容完后将原数组内容拷贝到新的数组中，空余部分填充缺省对象 UNSET。

> 为什么 InternalThreadLocalMap **以 index 为基准进行扩容**，而不是原数组长度呢？假设现在初始化了 70 个  FastThreadLocal，但是这些 FastThreadLocal 从来没有调用过 set() 方法，此时数组还是默认长度 32。当第  index = 70 的 FastThreadLocal 调用 set() 方法时，如果按原数组容量 32 进行扩容 2 倍后，还是无法填充  index = 70 的数据。所以使用 index 为基准进行扩容可以解决这个问题。

回到 setKnownNotUnset() 的主流程，threadLocalMap.addToVariablesToRemove():

```java
private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
    // variablesToRemoveIndex 是采用 static final 修饰的变量，
    // 在 FastThreadLocal 初始化时 variablesToRemoveIndex 被赋值为 0
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);// 取得 0 位元素
    Set<FastThreadLocal<?>> variablesToRemove;
    if (v == InternalThreadLocalMap.UNSET || v == null) {
        // 创建 FastThreadLocal 类型的 Set 集合
        variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
        threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
    } else {
        // 如果不是 UNSET，Set 集合已存在，直接强转获得 Set 集合
        variablesToRemove = (Set<FastThreadLocal<?>>) v;
    }
	// 将 FastThreadLocal 添加到 Set 集合中
    variablesToRemove.add(variable);
}
```

为什么 InternalThreadLocalMap 要在数组下标为 0 的位置存放一个 FastThreadLocal 类型的 Set 集合？实际上这与 set() 中的 remove() 有关：

```java
public final void remove() {
    remove(InternalThreadLocalMap.getIfSet());
}
// 获取当前 InternalThreadLocalMap
public static InternalThreadLocalMap getIfSet() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        return ((FastThreadLocalThread) thread).threadLocalMap();
    }
    return slowThreadLocalMap.get();
}
public final void remove(InternalThreadLocalMap threadLocalMap) {
    if (threadLocalMap == null) {
        return;
    }
	// 将 index 位置的元素覆盖为缺省对象 UNSET
    Object v = threadLocalMap.removeIndexedVariable(index);
    removeFromVariablesToRemove(threadLocalMap, this);

    if (v != InternalThreadLocalMap.UNSET) {
        try {
            onRemoval((V) v);// 空方法，用户可以继承实现
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
    }
}
// 取出 Set 集合，并删除当前 FastThreadLocal
private static void removeFromVariablesToRemove(
    InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {

    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);

    if (v == InternalThreadLocalMap.UNSET || v == null) {
        return;
    }

    @SuppressWarnings("unchecked")
    Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
    variablesToRemove.remove(variable);
}
```

InternalThreadLocalMap 会取出数组下标 0 位置的 Set 集合，然后删除当前 FastThreadLocal。至此，FastThreadLocal.set()已结束，现再看 FastThreadLocal.get() 

```java
public final V get() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }
	// 如果获取到的数组元素是缺省对象，执行初始化操作
    return initialize(threadLocalMap);
}
// 从数组中取出 index 位置的元素
public Object indexedVariable(int index) {
    Object[] lookup = indexedVariables;
    return index < lookup.length? lookup[index] : UNSET;
}
private V initialize(InternalThreadLocalMap threadLocalMap) {
    V v = null;
    try {
        v = initialValue();// 会调用用户重写的 initialValue 
    } catch (Exception e) {
        PlatformDependent.throwException(e);
    }

    threadLocalMap.setIndexedVariable(index, v);
    addToVariablesToRemove(threadLocalMap, this);
    return v;
}
```

- **FastThreadLocal 真的一定比 ThreadLocal 快吗**？答案是不一定的，只有使用FastThreadLocalThread 类型的线程才会更快，如果是普通线程反而会更慢。

- **FastThreadLocal 会浪费很大的空间吗**？虽然 FastThreadLocal  采用的空间换时间的思路，但是在 FastThreadLocal 设计之初就认为不会存在特别多的 FastThreadLocal  对象，而且在数据中没有使用的元素只是存放了同一个缺省对象的引用，并不会占用太多内存空间。

**FastThreadLocal 的优势**：

- **高效查找**。FastThreadLocal 在定位数据的时候可以直接根据数组下标 index 获取，时间复杂度 O(1)。而 JDK 原生的 ThreadLocal  在数据较多时哈希表很容易发生 Hash 冲突，线性探测法在解决 Hash  冲突时需要不停地向下寻找，效率较低。此外，FastThreadLocal 相比 ThreadLocal  数据扩容更加简单高效，FastThreadLocal 以 index 为基准向上取整到 2 的次幂作为扩容后容量，然后把原数据拷贝到新数组。而  ThreadLocal 由于采用的哈希表，所以在扩容后需要再做一轮 rehash。
- **安全性更高**。JDK 原生的  ThreadLocal 使用不当可能造成内存泄漏，只能等待线程销毁。在使用线程池的场景下，ThreadLocal  只能通过主动检测的方式防止内存泄漏，从而造成了一定的开销。然而 FastThreadLocal 不仅提供了 remove()  主动清除对象的方法，而且在线程池场景中 Netty 还封装了  FastThreadLocalRunnable，FastThreadLocalRunnable 最后会执行  FastThreadLocal.removeAll() 将 Set 集合中所有 FastThreadLocal 对象都清理掉，

---

### Netty源码（六）HashedWheelTimer 

HashedWheelTimer 是 Netty 中时间轮算法的实现类，关于时间轮算法这里就不在多说，参考之前的文章。

> Netty / Kafka 为何不使用 Timer、DelayQueue 和 ScheduledThreadPool 做定时任务？
>
> 问题就出在**时间复杂度**上，插入删除时间复杂度是O(logn)，那么假设频繁插入删除次数为 m，总的时间复杂度就是O(mlogn)

#### 1 上层接口

HashedWheelTimer 实现了接口 io.netty.util.Timer：

```java
public interface Timer {

    /**
     * Schedules the specified {@link TimerTask} for one-time execution after
     * the specified delay.
     *
     * @return a handle which is associated with the specified task
     *
     * @throws IllegalStateException       if this timer has been {@linkplain #stop() stopped} already
     * @throws RejectedExecutionException if the pending timeouts are too many and creating new timeout
     *                                    can cause instability in the system.
     */
    Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);

    /**
     * Releases all resources acquired by this {@link Timer} and cancels all
     * tasks which were scheduled but not executed yet.
     * 看来 Timer 持有 TimeOut 的集合
     * @return the handles associated with the tasks which were canceled by
     *         this method
     */
    Set<Timeout> stop();
}
```

Timer 接口提供了两个方法，分别是创建任务 newTimeout() 和停止所有未执行任务 stop()。

Timer 可以认为是上层的时间轮调度器，通过 newTimeout() 方法可以提交一个任务 TimerTask，并返回一个 Timeout。TimerTask 和 Timeout 是两个接口类：

```java
public interface TimerTask {

    /**
     * Executed after the delay specified with
     * {@link Timer#newTimeout(TimerTask, long, TimeUnit)}.
     *
     * @param timeout a handle which is associated with this task
     */
    void run(Timeout timeout) throws Exception;
}
```

```java
public interface Timeout {

    /**
     * Returns the {@link Timer} that created this handle.
     */
    Timer timer();

    /**
     * Returns the {@link TimerTask} which is associated with this handle.
     */
    TimerTask task();

    /**
     * Returns {@code true} if and only if the {@link TimerTask} associated
     * with this handle has been expired.
     */
    boolean isExpired();

    /**
     * Returns {@code true} if and only if the {@link TimerTask} associated
     * with this handle has been cancelled.
     */
    boolean isCancelled();

    /**
     * Attempts to cancel the {@link TimerTask} associated with this handle.
     * If the task has been executed or cancelled already, it will return with
     * no side effect.
     *
     * @return True if the cancellation completed successfully, otherwise false
     */
    boolean cancel();
}
```

Timeout 持有 Timer 和 TimerTask 的引用，而且通过 Timeout 接口可以执行取消任务的操作。Timer、Timeout 和 TimerTask 之间的关系如下图所示：

![CgpVE1_okNGAJA8SAAJSJkBDij0471](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgpVE1_okNGAJA8SAAJSJkBDij0471.png)

#### 2 使用示例

```java
public static void main(String[] args) {
    Timer timer = new HashedWheelTimer();
    System.out.println("---------------begin---------------" + new Date());
    Timeout timeout1 = timer.newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            System.out.println("timeout1: " + new Date());
        }
    }, 10 , TimeUnit.SECONDS);
    timer.newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            System.out.println("timeout2: " + new Date());
            // 任务二休眠5s
            Thread.sleep(5000);
        }
    },1, TimeUnit.SECONDS);
    timer.newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            System.out.println("timeout3: " + new Date());
        }
    },3, TimeUnit.SECONDS);
    // 取消 timeout1
    if (!timeout1.isExpired()) {
        timeout1.cancel();
    }
    System.out.println("---------------end---------------" + new Date());
}
```

```java
---------------begin---------------Fri Mar 26 10:16:16 CST 2021
---------------end---------------Fri Mar 26 10:16:16 CST 2021
timeout2: Fri Mar 26 10:16:17 CST 2021
timeout3: Fri Mar 26 10:16:22 CST 2021
```

输出如上所示，添加任务是异步的，任务10s后被执行，也就是会在10:16:26时执行，然而，我们把它取消了，所以没有执行。

任务2设置1秒后执行，任务3设置3秒后执行。理论上 timeout3 时间应该为 10:16:19，但却是10:16:22，其原因是由于任务二休眠5s导致。如果任务2不休眠，那么就是正常时间。说明任务的是**串行执行**，会被阻塞。当一个任务执行的时间过长，会影响后续任务的调度和执行，很可能**产生任务堆积**的情况。

#### 3 源码

##### 3.1  构造方法

HashedWheelTimer 有一堆构造方法，其底层都是调用了一个方法：

```java
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
        long maxPendingTimeouts) {
	// 校验参数代码省略

    // Normalize ticksPerWheel to power of two and initialize the wheel.
    wheel = createWheel(ticksPerWheel);// 创建时间轮的环形数组结构
    mask = wheel.length - 1;// 用于快速取模的掩码

    // Convert tickDuration to nanos.
    long duration = unit.toNanos(tickDuration);// 转换成纳秒处理

    // Prevent overflow.
    if (duration >= Long.MAX_VALUE / wheel.length) {
        throw new IllegalArgumentException(String.format(
                "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                tickDuration, Long.MAX_VALUE / wheel.length));
    }

    if (duration < MILLISECOND_NANOS) {
        logger.warn("Configured tickDuration {} smaller then {}, using 1ms.",
                    tickDuration, MILLISECOND_NANOS);
        this.tickDuration = MILLISECOND_NANOS;
    } else {
        this.tickDuration = duration;
    }
	// 创建工作线程
    workerThread = threadFactory.newThread(worker);
	// 是否开启内存泄漏检测
    leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;
	// 最大允许等待任务数，HashedWheelTimer 中任务超出该阈值时会抛出异常，如果为0或负数则不限制，这里为 -1
    this.maxPendingTimeouts = maxPendingTimeouts;
	// INSTANCE_COUNT_LIMIT 为 64 ，如果 HashedWheelTimer 的实例数超过 64，会打印错误日志
    if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
        WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
        reportTooManyInstances();
    }
}
```

构造方法中参数如下：

- **threadFactory**，线程池，但仅创建了一个线程；
- **tickDuration**，时针每次 tick 的时间，相当于时针间隔多久走到下一个 slot；
- **unit**，表示 tickDuration 的时间单位；
- **ticksPerWheel**，时间轮上一共有多少个 slot，默认 512 个。分配的 slot 越多，占用的内存空间就越大；
- **leakDetection**，是否开启内存泄漏检测；
- **maxPendingTimeouts**，最大允许等待任务数。No maximum pending timeouts limit is assumed if this value is 0 or negative

createWheel：

```java
private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
    // 校验参数代码略
    ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);// 计算数组容量
    HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
    for (int i = 0; i < wheel.length; i ++) {
        wheel[i] = new HashedWheelBucket();
    }
    return wheel;
}
// 此方法是取得 ticksPerWheel 最小的二次幂，比如 7 取得 8，和之前一个方法类似，但不清楚此处为何不进行多次位运算。
private static int normalizeTicksPerWheel(int ticksPerWheel) {
    int normalizedTicksPerWheel = 1;
    while (normalizedTicksPerWheel < ticksPerWheel) {
        normalizedTicksPerWheel <<= 1;
    }
    return normalizedTicksPerWheel;
}
// HashedWheelBucket 是一个双向链表
private static final class HashedWheelBucket {
    // Used for the linked-list datastructure
    private HashedWheelTimeout head;
    private HashedWheelTimeout tail;
    // 其余略
}
```

源码（五）曾经介绍过 expandIndexedVariableTableAndSet，其作用与 normalizeTicksPerWheel 一致。这里 normalizeTicksPerWheel 这么实现有些不妥，不过初始化调用一次，影响不大。

时间轮的创建就是为了创建 HashedWheelBucket 数组，数组便是整个时间轮，而每个 HashedWheelBucket 表示时间轮中一个 slot。HashedWheelBucket 是双向链表，可以实现不同方向进行链表遍历。

![CgqCHl_okUGATnXpAAPdCRAt-n0348](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgqCHl_okUGATnXpAAPdCRAt-n0348.png)

##### 3.2  添加任务

HashedWheelTimer 初始化完成后，如何向 HashedWheelTimer 添加任务呢？我们自然想到 HashedWheelTimer 提供的 newTimeout() 方法。

```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    // 省略参数校验代码
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();

    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
        pendingTimeouts.decrementAndGet();
        throw new RejectedExecutionException("Number of pending timeouts ("
            + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
            + "timeouts (" + maxPendingTimeouts + ")");
    }
	// 1. 懒启动
    start();

    // Add the timeout to the timeout queue which will be processed on the next tick.
    // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

    // Guard against overflow.
    if (delay > 0 && deadline < 0) {
        deadline = Long.MAX_VALUE;
    }
    // 2. 创建定时任务
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    // 3. 添加任务到 Mpsc Queue, 想借助 Mpsc Queue 保证多线程向时间轮添加任务的线程安全性。
    timeouts.add(timeout);
    return timeout;
}
```

HashedWheelTimer 的工作线程采用了懒启动的方式，不需要用户显示调用。这样做的好处是在时间轮中没有任务时，可以避免工作线程空转而造成性能损耗，来看 start():

```java
public void start() {
    switch (WORKER_STATE_UPDATER.get(this)) {// 通过 CAS 操作获取工作线程的状态
        case WORKER_STATE_INIT:
            // 如果没有启动，再次通过 CAS 操作更改工作线程状态
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                // 启动的过程是直接调用的 Thread#start() 
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:// 已经启动，则直接跳过
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    // Wait until the startTime is initialized by the worker.
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```

而后将任务添加到Mpsc Queue中，至于为何，下节再说。

##### 3.3 工作线程 Worker

工作线程 Worker 是时间轮的核心引擎，随着时针的转动，到期任务的处理都由 Worker 处理完成。

```java
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    private long tick;

    @Override
    public void run() {
        // Initialize the startTime.
        startTime = System.nanoTime();
        if (startTime == 0) {
            // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
            startTime = 1;
        }

        // Notify the other threads waiting for the initialization at start().
        startTimeInitialized.countDown();

        do {
            // 1. 计算下次 tick 的时间, 然后sleep 到下次 tick
            final long deadline = waitForNextTick();
            // 可能因为溢出或者线程中断，造成 deadline <= 0
            if (deadline > 0) {
                // 2. 获取当前 tick 在 HashedWheelBucket 数组中对应的下标
                int idx = (int) (tick & mask);
                // 3. 移除被取消的任务
                processCancelledTasks();
                HashedWheelBucket bucket = wheel[idx];
                // 4. 从 Mpsc Queue 中取出任务加入对应的 slot 中
                transferTimeoutsToBuckets();
                // 5. 执行到期的任务
                bucket.expireTimeouts(deadline);
                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
		// 时间轮退出后，取出 slot 中未执行且未被取消的任务，并加入未处理任务列表，以便 stop() 方法返回
        // Fill the unprocessedTimeouts so we can return them from stop() method.
        for (HashedWheelBucket bucket: wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        for (;;) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        processCancelledTasks();
    }
    // 其余略
}
```

其5个主要步骤在注释中写了，首先看看 waitForNextTick() 

```java
private long waitForNextTick() {
    long deadline = tickDuration * (tick + 1);

    for (;;) {
        // 根据 tickDuration 可以推算出下一次 tick 的 deadline
        final long currentTime = System.nanoTime() - startTime;
        // deadline 减去当前时间就可以得到需要 sleep 的等待时间
        long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

        if (sleepTimeMs <= 0) {
            if (currentTime == Long.MIN_VALUE) {
                return -Long.MAX_VALUE;
            } else {
                return currentTime;
            }
        }

        // Check if we run on windows, as if thats the case we will need
        // to round the sleepTime as workaround for a bug that only affect
        // the JVM if it runs on windows.
        //
        // See https://github.com/netty/netty/issues/356
        if (PlatformDependent.isWindows()) {
            sleepTimeMs = sleepTimeMs / 10 * 10;
        }

        try {
            Thread.sleep(sleepTimeMs);
        } catch (InterruptedException ignored) {
            if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                return Long.MIN_VALUE;
            }
        }
    }
}
```

继续看第二步 获取当前 tick 在 HashedWheelBucket 数组中对应的下标，就是位运算 做取模操作。

第三步 processCancelledTasks() 处理被取消的任务，就是去链表中删除。

第四步 transferTimeoutsToBuckets()  从 Mpsc Queue 中取出任务加入对应的 HashedWheelBucket 中：

```java
private void transferTimeoutsToBuckets() {
    // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
    // adds new timeouts in a loop.
    for (int i = 0; i < 100000; i++) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            // all processed
            break;
        }
        if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
            // Was cancelled in the meantime.
            continue;
        }
		// 计算任务需要经过多少个 tick
        long calculated = timeout.deadline / tickDuration;
        // 计算任务需要在时间轮中经历的圈数 remainingRounds
        timeout.remainingRounds = (calculated - tick) / wheel.length;
		// 如果任务在 timeouts 队列里已经过了执行时间, 那么会加入当前 HashedWheelBucket 中
        final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
        int stopIndex = (int) (ticks & mask);

        HashedWheelBucket bucket = wheel[stopIndex];
        bucket.addTimeout(timeout);
    }
}
```

> 每次时针 tick 最多只处理 100000 个任务，一方面避免取任务的操作耗时过长，另一方面为了防止执行太多任务造成 Worker 线程阻塞。计算出任务需要经过多少次 tick 才能开始执行以及需要在时间轮中转动圈数 remainingRounds，并把remainingRounds 记录在 HashedWheelTimeout 中，假如有一个任务执行的时间特别长，任务在 timeouts 队列里已经过了执行时间，Worker 会将这些任务直接加入当前HashedWheelBucket 中，故过期的任务并不会被遗漏。

第五步  expireTimeouts() 执行到期的任务

```java
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;

    // process all timeouts
    while (timeout != null) {
        HashedWheelTimeout next = timeout.next;
        // 时间到了，执行任务
        if (timeout.remainingRounds <= 0) {
            next = remove(timeout);
            if (timeout.deadline <= deadline) {
                timeout.expire();
            } else {
                // The timeout was placed into a wrong slot. This should never happen.
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
        } else if (timeout.isCancelled()) {// 被取消
            next = remove(timeout);
        } else {// 下一圈
            timeout.remainingRounds --;// 未到执行时间，remainingRounds(转动圈数) 减 1
        }
        timeout = next;
    }
}
```

##### 3.4 stop()

```java
@Override
public Set<Timeout> stop() {
    // Worker 线程无法停止时间轮
    if (Thread.currentThread() == workerThread) {
        throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() +
                        ".stop() cannot be called from " +
                        TimerTask.class.getSimpleName());
    }
 	// 1. 尝试通过 CAS 操作将工作线程的状态更新为 SHUTDOWN 状态
    if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
        // workerState can be 0 or 2 at this moment - let it always be 2.
        if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
            INSTANCE_COUNTER.decrementAndGet();
            if (leak != null) {
                boolean closed = leak.close(this);
                assert closed;
            }
        }

        return Collections.emptySet();
    }

    try {
        boolean interrupted = false;
        while (workerThread.isAlive()) {
            // 2. 中断 Worker 线程
            workerThread.interrupt();
            try {
                workerThread.join(100);
            } catch (InterruptedException ignored) {
                interrupted = true;
            }
        }

        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    } finally {
        INSTANCE_COUNTER.decrementAndGet();
        if (leak != null) {
            boolean closed = leak.close(this);
            assert closed;
        }
    }
    // 3.返回未处理任务的列表
    return worker.unprocessedTimeouts();
}
```

> 当前线程是 Worker 线程，它是不能发起停止时间轮的操作的，是为了防止有定时任务发起停止时间轮的恶意操作

至此，HashedWheelTimer 的实现原理我们已经分析完了：

- **HashedWheelTimeout**，任务的封装类，包含任务的到期时间 deadline、需要经历的圈数 remainingRounds 等属性。

- **HashedWheelBucket**，相当于时间轮的每个 slot，内部采用双向链表保存了当前需要执行的 HashedWheelTimeout 列表。

- **Worker**，HashedWheelTimer 的核心工作引擎，负责处理定时任务。

#### 4 拓展

我们说到时间轮时，说道了分层时间轮，不过 Netty 并没有使用分层时间轮，而且执行任务默认只有一个线程，如果任务堵塞，就会造成其他任务执行时间不准。再有会有很多无用的空推进，比如 A 任务 1s 后执行，B 任务 6 小时之后执行。

其实 Kafka 中也用到时间轮，而且一般问到时间轮算法，必会问到在 Netty 和 kafka 中的应用。

Kafka 的时间轮也是采用环形数组存储定时任务，数组中的每个 slot 代表一个 Bucket，每个 Bucket 保存了定时任务列表 TimerTaskList，TimerTaskList 同样采用双向链表的结构实现。

首先为了解决**空推进**问题，Kafka 借助 JDK 的 DelayQueue 来负责推进时间轮。DelayQueue 保存了时间轮中的每个 Bucket，并且根据 Bucket 的到期时间进行排序，最近的到期时间被放在 DelayQueue 的队头，如果时间没有到，那么 DelayQueue 会一直处于阻塞状态，从而解决空推进的问题。DelayQueue 只存放了 Bucket，Bucket 的数量并不多，相比空推进带来的影响是利大于弊的。

为了解决任务时间跨度很大的问题，Kafka 引入了**层级时间轮**，如下图所示。当任务的 deadline **超出当前所在层的时间轮表示范围**时，就会尝试将任务添加到上一层时间轮中，跟钟表的时针、分针、秒针的转动规则是同一个道理。

![Ciqc1F_okdOAION1AALmn-njKt8140](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Ciqc1F_okdOAION1AALmn-njKt8140.png)

第一层时间轮每个时间格为 1ms，整个时间轮的跨度为 20ms；第二层时间轮每个时间格为 20ms，整个时间轮跨度为 400ms；第三层时间轮每个时间格为 400ms，整个时间轮跨度为 8000ms。每一层时间轮都有自己的指针，每层时间轮走完一圈后，上层时间轮也会相应推进一格。

假设现在有一个任务到期时间是 450ms 之后，应该放在第三层时间轮的第一格。随着时间的流逝，当指针指向该时间格时，发现任务到期时间还有 50ms，这里就涉及时间轮降级的操作，它会将任务重新提交到时间轮中。此时发现第一层时间轮整体跨度不够，需要放在第二层时间轮中第三格。当时间再经历 40ms 之后，该任务又会触发一次降级操作，放入到第一层时间轮，最后等到 10ms 后执行任务。

#### 5 总结

HashedWheelTimer 可以高效处理大量定时任务，但也是有缺陷的：

- 如果长时间没有到期任务，那么会存在时间轮空推进的现象。

- 只适用于处理耗时较短的任务，由于 Worker 是单线程的，如果一个任务执行的时间过长，会造成 Worker 线程阻塞。

- 相比传统定时器的实现方式，内存占用较大。

> 参考
>
> [面试官：知道时间轮算法吗？在Netty和Kafka中如何应用的？](https://mp.weixin.qq.com/s/lBJs8mQpPJGdleLqtGS-dw)


---

### Netty源码（七） Mpsc Queue

> Mpsc :Multi Producer Single Consumer，多生产者单消费者

之前我们多次讲到 Mpsc Queue，比如 NioEventLoop 线程以及时间轮 HashedWheelTimer 的任务队列中都出现了 Mpsc Queue 的身影。现在就来讲讲这个高性能无锁队列。

#### 1 JDK原生并发队列

JDK 并发队列按照实现方式可以分为阻塞队列和非阻塞队列两种类型，阻塞队列是基于锁实现的，非阻塞队列是基于 CAS 操作实现的。JDK 中包含多种阻塞和非阻塞的队列实现，如下图：

![Cip5yF_sPcmAaDp3AAIqvvYOtM0974](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Cip5yF_sPcmAaDp3AAIqvvYOtM0974.png)

##### 1.1 阻塞队列

阻塞队列在队列为空或者队列满时，都会发生阻塞。阻塞队列自身是线程安全的，使用者无需关心线程安全问题，降低了多线程开发难度。

![CgpVE1_sPeSAZ3hUAAESfy4p6LY171](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgpVE1_sPeSAZ3hUAAESfy4p6LY171.png)

阻塞队列主要分为以下几种：

- **ArrayBlockingQueue**：最基础且开发中最常用的阻塞队列，底层采用**数组实现的有界队列**，初始化需要指定队列的容量。ArrayBlockingQueue 是如何保证线程安全的呢？它内部是使用了一个重入锁 ReentrantLock，并搭配 notEmpty、notFull 两个条件变量 Condition 来控制并发访问。从队列读取数据时，如果队列为空，那么会阻塞等待，直到队列有数据了才会被唤醒。如果队列已经满了，也同样会进入阻塞状态，直到队列有空闲才会被唤醒。
- **LinkedBlockingQueue**：内部采用的数据结构是**链表**，队列的长度可以是有界或者无界的，初始化不需要指定队列长度，默认是 Integer.MAX_VALUE。LinkedBlockingQueue 内部使用了 takeLock、putLock两个重入锁 ReentrantLock，以及 notEmpty、notFull 两个条件变量 Condition 来控制并发访问。采用读锁和写锁的好处是可以避免读写时相互竞争锁的现象，所以相比于 ArrayBlockingQueue，LinkedBlockingQueue 的性能要更好。
- **PriorityBlockingQueue**：采用**最小堆实现的优先级队列**，队列中的元素按照优先级进行排列，每次出队都是返回优先级最高的元素。PriorityBlockingQueue 内部是使用了一个 ReentrantLock 以及一个条件变量 Condition notEmpty 来控制并发访问，不需要 notFull 是因为 PriorityBlockingQueue 是无界队列，所以每次 put 都不会发生阻塞。PriorityBlockingQueue 底层的最小堆是采用数组实现的，当元素个数大于等于最大容量时会触发扩容，在扩容时会先释放锁，保证其他元素可以正常出队，然后使用 CAS 操作确保只有一个线程可以执行扩容逻辑。
- **DelayQueue**：一种**支持延迟获取元素**的阻塞队列，常用于缓存、定时任务调度等场景。DelayQueue 内部是采用优先级队列 PriorityQueue 存储对象。DelayQueue 中的每个对象都必须实现 Delayed 接口，并重写 compareTo 和 getDelay 方法。向队列中存放元素的时候必须指定延迟时间，只有延迟时间已满的元素才能从队列中取出。
- **SynchronizedQueue**：又称无缓冲队列。比较特别的是 SynchronizedQueue **内部不会存储元素**。与 ArrayBlockingQueue、LinkedBlockingQueue 不同，SynchronizedQueue 直接使用 CAS 操作控制线程的安全访问。其中 put 和 take 操作都是阻塞的，每一个 put 操作都必须阻塞等待一个 take 操作，反之亦然。所以 SynchronizedQueue 可以理解为生产者和消费者配对的场景，双方必须互相等待，直至配对成功。在 JDK 的线程池 Executors.newCachedThreadPool 中就存在 SynchronousQueue 的运用，对于新提交的任务，如果有空闲线程，将重复利用空闲线程处理任务，否则将新建线程进行处理。
- **LinkedTransferQueue**：一种特殊的无界阻塞队列，可以看作 LinkedBlockingQueues、SynchronousQueue（公平模式）、ConcurrentLinkedQueue 的合体。与 SynchronousQueue 不同的是，LinkedTransferQueue 内部可以存储实际的数据，当执行 put 操作时，**如果有等待线程，那么直接将数据交给对方，否则放入队列中**。与 LinkedBlockingQueues 相比，LinkedTransferQueue 使用 CAS 无锁操作进一步提升了性能。

##### 1.2 非阻塞队列

非阻塞队列不需要通过加锁的方式对线程阻塞，并发性能更好：

- **ConcurrentLinkedQueue**：它是一个采用双向链表实现的无界并发非阻塞队列，它属于 LinkedQueue 的安全版本。ConcurrentLinkedQueue 内部采用 CAS 操作保证线程安全，这是非阻塞队列实现的基础，相比 ArrayBlockingQueue、LinkedBlockingQueue 具备较高的性能。
- **ConcurrentLinkedDeque**：也是一种采用双向链表结构的无界并发非阻塞队列。与 ConcurrentLinkedQueue 不同的是，ConcurrentLinkedDeque 属于双端队列，它同时支持 FIFO 和 FILO 两种模式，可以从队列的头部插入和删除数据，也可以从队列尾部插入和删除数据，适用于多生产者和多消费者的场景。

#### 2 高性能无锁队列

第三方框架提供的高性能无锁队列非常出名的有 Disruptor 和 JCTools。

**Disruptor** 是 LMAX 公司开发的一款高性能无锁队列，我们平时常称它为 RingBuffer，其设计初衷是为了解决内存队列的延迟问题。Disruptor 内部采用环形数组和 CAS 操作实现，性能非常优越。为什么 Disruptor 的性能会比 JDK 原生的无锁队列要好呢？环形数组可以复用内存，减少分配内存和释放内存带来的性能损耗。而且数组可以设置长度为 2 的次幂，直接通过位运算加快数组下标的定位速度。此外，Disruptor 还解决了伪共享问题，对 CPU Cache 更加友好。Disruptor 已经开源，详细可查阅 Github 地址 https://github.com/LMAX-Exchange/disruptor。

**JCTools** 也是一个开源项目，Github 地址为 https://github.com/JCTools/JCTools。JCTools 是适用于 JVM 并发开发的工具，主要提供了一些 JDK 的并发数据结构，例如非阻塞 Map、非阻塞 Queue 等。其中非阻塞队列可以分为四种类型，可以根据不同的场景选择使用。

- Spsc 单生产者单消费者；
- Mpsc 多生产者单消费者；
- Spmc 单生产者多消费者；
- Mpmc 多生产者多消费者。

Netty 中就是直接引入了 JCTools 的 Mpsc Queue

#### 3 Mpsc Queue 基础

Mpsc Queue 可以保证多个生产者同时访问队列是线程安全的，而且同一时刻只允许一个消费者从队列中读取数据。Netty Reactor 线程中任务队列 taskQueue 必须满足多个生产者可以同时提交任务。

Mpsc Queue 有多种的实现类，例如 MpscArrayQueue、MpscUnboundedArrayQueue、MpscChunkedArrayQueue 等，首先来看下MpscArrayQueue

![image-20210408143945759](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/image-20210408143945759.png)

使用 ctrl + h 查看类的继承关系，发现它的父类十分多，名字格式很多是 MpscArrayQueueXxxPad 以及 MpscArrayQueueXxxField 。

们自顶向下，将所有类的字段合并在一起：

```java
// ConcurrentCircularArrayQueueL0Pad
long p01, p02, p03, p04, p05, p06, p07;
long p10, p11, p12, p13, p14, p15, p16, p17;
// ConcurrentCircularArrayQueue
protected final long mask;
protected final E[] buffer;
// MpmcArrayQueueL1Pad
long p00, p01, p02, p03, p04, p05, p06, p07;
long p10, p11, p12, p13, p14, p15, p16;
// MpmcArrayQueueProducerIndexField
private volatile long producerIndex;
// MpscArrayQueueMidPad
long p01, p02, p03, p04, p05, p06, p07;
long p10, p11, p12, p13, p14, p15, p16, p17;
// MpscArrayQueueProducerLimitField
private volatile long producerLimit;
// MpscArrayQueueL2Pad
long p00, p01, p02, p03, p04, p05, p06, p07;
long p10, p11, p12, p13, p14, p15, p16;
// MpscArrayQueueConsumerIndexField
protected long consumerIndex;
// MpscArrayQueueL3Pad
long p01, p02, p03, p04, p05, p06, p07;
long p10, p11, p12, p13, p14, p15, p16, p17;
```

MpscArrayQueueXxxPad  类中使用了大量 long 类型的变量, Mpsc Queue 之所以填充这些无意义的变量，是为了解决**伪共享**（false sharing）问题。

> Disruptor 也使用了类似的填充方法

##### 伪共享

在计算机组成中，CPU 的运算速度比内存高出几个数量级，为了 CPU 能够更高效地与内存进行交互，在 CPU 和内存之间设计了多层缓存机制，如下图所示

![CgqCHl_sPgiAL4s1AANDqZ3wy8M196](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/CgqCHl_sPgiAL4s1AANDqZ3wy8M196.png)

一般来说，CPU 会分为三级缓存，分别为**L1 一级缓存**、**L2 二级缓存**和**L3 三级缓存**。越靠近 CPU 的缓存，速度越快，但是缓存的容量也越小。所以从性能上来说，L1 > L2 > L3，容量方面 L1 < L2 < L3。CPU 读取数据时，首先会从 L1 查找，如果未命中则继续查找 L2，如果还未能命中则继续查找 L3，最后还没命中的话只能从内存中查找，读取完成后再将数据逐级放入缓存中。此外，多线程之间共享一份数据的时候，需要其中一个线程将数据写回主存，其他线程访问主存数据。

引入多级缓存是为了能够让 CPU 利用率最大化。如果你在做频繁的 CPU 运算时，需要尽可能将数据保持在缓存中。那么 CPU 从内存中加载数据的时候，是如何提高缓存的利用率的呢？这就涉及缓存行（Cache Line）的概念，Cache Line 是 CPU 缓存可操作的最小单位，CPU 缓存由若干个 Cache Line 组成。Cache Line 的大小与 CPU 架构有关，在目前主流的 64 位架构下，Cache Line 的大小通常为 64 Byte。Java 中一个 long 类型是 8 Byte，所以一个 Cache Line 可以存储 8 个 long 类型变量。CPU 在加载内存数据时，会将相邻的数据一同读取到 Cache Line 中，因为相邻的数据未来被访问的可能性最大，这样就可以避免 CPU 频繁与内存进行交互了。

伪共享问题是如何发生的呢？

![Ciqc1F_sPh-AQ8DnAARFoCjQlXQ789](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Ciqc1F_sPh-AQ8DnAARFoCjQlXQ789.png)

假设变量 A、B、C、D 被加载到同一个 Cache Line，它们会被高频地修改。当线程 1 在 CPU Core1 中中对变量 A 进行修改，修改完成后 CPU Core1 会通知其他 CPU Core 该缓存行已经失效。然后线程 2 在 CPU Core2 中对变量 C 进行修改时，发现 Cache line 已经失效，此时 CPU Core1 会将数据重新写回内存，CPU Core2 再从内存中读取数据加载到当前 Cache line 中。

由此可见，如果同一个 Cache line 被越多的线程修改，那么造成的写竞争就会越激烈，数据会频繁写入内存，导致性能浪费。

伪共享问题的表现是：并发的修改在一个缓存行中的多个独立变量，看起来是并发执行的，但实际在CPU处理的时候，是串行执行的，并发的性能大打折扣。就像上面的 cpu core1修改A，cpu core2修改C，这本应该是并发执行，但是这两个变量在一个缓存行，由于[缓存一致性](https://blog.csdn.net/muxiqingyang/article/details/6615199)，却成为了串行操作。

解决方案有两种，一是 填充法，二是使用 Contented 注解

```java
public class FalseSharingPadding {
    protected long p1, p2, p3, p4, p5, p6, p7;
    protected volatile long value = 0L;
    protected long p9, p10, p11, p12, p13, p14, p15;
}
```

从上述代码中可以看出，变量 value 前后都填充了 7 个 long 类型的变量。这样不论在什么情况下，都可以保证在多线程访问 value 变量时，value 与其他不相关的变量处于不同的 Cache Line，如下图所示。

![Ciqc1F_sPhSAPI-iAAP2dKAqN5g543](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Ciqc1F_sPhSAPI-iAAP2dKAqN5g543.png)

伪共享问题一般是非常隐蔽的，在实际开发的过程中，并不是项目中所有地方都需要花费大量的精力去优化伪共享问题。CPU Cache 的填充本身也是比较珍贵的,所以伪共享优化不是我们研究的重点。

#### 4 Mpsc Queue 源码

##### 4.1 使用范例

```java
public class MpscArrayQueueTest {
    public static final MpscArrayQueue<String> MPSC_ARRAY_QUEUE = new MpscArrayQueue<>(2);
    public static void main(String[] args) {
        for (int i = 1; i <= 2; i++) {
            int index = i;
            new Thread(() -> MPSC_ARRAY_QUEUE.offer("data" + index), "thread" + index).start();
        }
        try {
            Thread.sleep(1000L);
            MPSC_ARRAY_QUEUE.add("data3"); // 入队操作，队列满则抛出异常
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("队列大小：" + MPSC_ARRAY_QUEUE.size() + ", 队列容量：" + MPSC_ARRAY_QUEUE.capacity());
        System.out.println("出队：" + MPSC_ARRAY_QUEUE.remove()); // 出队操作，队列为空则抛出异常
        System.out.println("出队：" + MPSC_ARRAY_QUEUE.poll()); // 出队操作，队列为空则返回 NULL
    }
}
```

```java
java.lang.IllegalStateException: Queue full
	at java.util.AbstractQueue.add(AbstractQueue.java:98)
	at MpscArrayQueueTest.main(MpscArrayQueueTest.java:17)
队列大小：2, 队列容量：2
出队：data1
出队：data2
Disconnected from the target VM, address: '127.0.0.1:58005', transport: 'socket'
```

##### 4.2 重要属性

```java
// ConcurrentCircularArrayQueue
protected final long mask; // 计算数组下标的掩码
protected final E[] buffer; // 存放队列数据的数组
// MpmcArrayQueueProducerIndexField
private volatile long producerIndex; // 生产者的索引
// MpscArrayQueueProducerLimitField
private volatile long producerLimit; // 生产者索引的最大值
// MpscArrayQueueConsumerIndexField
protected long consumerIndex; // 消费者索引
```

##### 4.3 offer()

```java
public boolean offer(E e) {
    if (null == e) {
        throw new NullPointerException();
    } else {
        long mask = this.mask;
        // 获取生产者索引最大限制
        long producerLimit = this.lvProducerLimit();

        long pIndex;
        long offset;
        do {
            // 获取生产者索引
            pIndex = this.lvProducerIndex();
            if (pIndex >= producerLimit) {
                // 获取消费者索引
                offset = this.lvConsumerIndex();
                producerLimit = offset + mask + 1L;
                if (pIndex >= producerLimit) {
                    return false;// 队列已满
                }
			   // 更新 producerLimit
                this.soProducerLimit(producerLimit);
            }
             
        } // CAS 更新生产者索引，更新成功则退出，说明当前生产者已经占领索引值
        while(!this.casProducerIndex(pIndex, pIndex + 1L));
		// 计算生产者索引在数组中下标
        offset = calcElementOffset(pIndex, mask);
        // 向数组中放入数据
        UnsafeRefArrayAccess.soElement(this.buffer, offset, e);
        return true;
    }
}
```

这个方法还是难以理解的的，我们还是结合上面的使用范例来理解吧。

在初始化状态，producerLimit 与队列的容量是相等的，producerLimit = capacity = 2，而 producerIndex = consumerIndex = 0。接下来 Thread1 和 Thread2 并发向 MpscArrayQueue 中存放数据，如下图所示

![Ciqc1F_sPjqAPdT3AAMJXNU1SQE737](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/picgo/Ciqc1F_sPjqAPdT3AAMJXNU1SQE737.png)

两个线程此时拿到的 producerIndex 都是 0，是小于 producerLimit 的。此时两个线程都会尝试使用 CAS 操作更新 producerIndex，其中必然有一个是成功的，另外一个是失败的。假设 Thread1 执行 CAS 操作成功，那么 Thread2 失败后就会重新更新 producerIndex。Thread1 更新后 producerIndex 的值为 1，由于 producerIndex 是 volatile 修饰的，更新后立刻对 Thread2 可见。有一点需要注意的是，当前线程更新后的值是被其他线程使用，也就是线程1 CAS操作成功，producerIndex 变为1，但是线程1使用的是 pIndex = 0 。线程2使用的时候 producerIndex  变为 2， pIndex = 1 。

然后就是将数据写入数组中：

```java
public static <E> void soElement(E[] buffer, long offset, E e) {
    UnsafeAccess.UNSAFE.putOrderedObject(buffer, offset, e);
}
```

putOrderedObject() 和 putObject() 都可以用于更新对象的值，但是 putOrderedObject() 并不会立刻将数据更新到内存中，并把其他 Cache Line 置为失效。putOrderedObject() 使用的是 LazySet **延迟更新机制**，所以性能方面 putOrderedObject() 要比 putObject() 高很多。

> Java 中有四种类型的内存屏障，分别为 LoadLoad、StoreStore、LoadStore 和 StoreLoad。putOrderedObject() 使用了 **StoreStore** Barrier，对于 Store1，StoreStore，Store2 这样的操作序列，在 Store2 进行写入之前，会保证 Store1 的写操作对其他处理器可见。
>
> LazySet 机制是有代价的，就是写操作结果有纳秒级的延迟，不会立刻被其他线程以及自身线程可见。因为在 Mpsc Queue 的使用场景中，**多个生产者只负责写入数据**，并没有写入之后立刻读取的需求，所以使用 LazySet 机制是没有问题的，只要 StoreStore Barrier 保证多线程写入的顺序即可。

 do-while 循环内，为什么**需要两次** if(pIndex >= producerLimit) ?

当生产者索引大于 producerLimit 阈值时，可能存在两种情况：producerLimit 缓存值过期了或者队列已经满了。所以此时我们需要读取最新的消费者索引 consumerIndex，之前读取过的数据位置都可以被重复使用，重新做一次 producerLimit 计算，然后再做一次 if(pIndex >= producerLimit) 判断，如果生产者索引还是大于 producerLimit 阈值，说明队列的真的满了。

MpscArrayQueue 采用了 UNSAFE.getLongVolatile() 方法保证**获取消费者索引** consumerIndex 的准确性。getLongVolatile() 使用了 **StoreLoad** Barrier，对于 Store1，StoreLoad，Load2 的操作序列，在 Load2 以及后续的读取操作之前，都会保证 Store1 的写入操作对其他处理器可见。但是StoreLoad 是四种内存屏障**开销最大**的，所以判断了只有当 pIndex >= producerLimit 才会执行 getLongVolatile() .,假设我们的消费速度和生产速度比较均衡的情况下，差不多走完一圈数组才需要获取一次消费者索引.

##### 4.4 poll()

```java
public E poll() {
    // 消费者索引 consumerIndex
    long cIndex = this.lpConsumerIndex();
    // 计算数组对应的偏移量
    long offset = this.calcElementOffset(cIndex);
    E[] buffer = this.buffer;
    // 取出数组中 offset 对应的元素
    E e = UnsafeRefArrayAccess.lvElement(buffer, offset);
    if (null == e) {
        if (cIndex == this.lvProducerIndex()) {// 队列为空
            return null;
        }

        do {
            e = UnsafeRefArrayAccess.lvElement(buffer, offset);
        } while(e == null);// 等待生产者填充元素
    }
	// 消费成功后将当前位置置为 NULL
    UnsafeRefArrayAccess.spElement(buffer, offset, (Object)null);
    // 更新 consumerIndex 到下一个位置
    this.soConsumerIndex(cIndex + 1L);
    return e;
}
// UnsafeRefArrayAccess
public static <E> E lvElement(E[] buffer, long offset) {
    return UnsafeAccess.UNSAFE.getObjectVolatile(buffer, offset);
}
public static <E> void spElement(E[] buffer, long offset, E e) {
    UnsafeAccess.UNSAFE.putObject(buffer, offset, e);
}
protected void soConsumerIndex(long newValue) {
    UNSAFE.putOrderedLong(this, C_INDEX_OFFSET, newValue);
}
```

获取数组元素的时候同样使用了 UNSAFE 系列方法，getObjectVolatile() 方法则使用的是 **LoadLoad**  Barrier，对于 Load1，LoadLoad，Load2 操作序列，在 Load2 以及后续读取操作之前，会保证 Load1 的读取操作执行完毕，所以 getObjectVolatile() 方法可以保证每次读取数据都可以从内存中拿到最新值。

当调用 lvElement() 方法获取到的元素为 NULL 时，有两种可能的情况：**队列为空**或者**生产者填充的元素还没有对消费者可见**。如果消费者索引 consumerIndex 等于生产者 producerIndex，说明队列为空。只要两者不相等，消费者需要等待生产者填充数据完毕。

putObject() 不会使用任何内存屏障，它会直接更新对象对应偏移量的值。而 putOrderedLong 与 putOrderedObject() 是一样的，都使用了 **StoreStore** Barrier，也是延迟更新 LazySet 机制。

#### 5 总结

- 通过大量填充 long 类型变量解决伪共享问题。
- 环形数组的容量设置为 2 的次幂，可以通过位运算快速定位到数组对应下标。
- 入队 offer() 操作中 producerLimit 的巧妙设计，大幅度减少了主动获取消费者索引 consumerIndex 的次数，性能提升显著。
- 入队和出队操作中都大量使用了 UNSAFE 系列方法，针对生产者和消费者的场景不同，使用的 UNSAFE 方法也是不一样的。Jctools 在底层操作的运用上也是有的放矢，把性能发挥到极致。

