---
title: Dubbo中对线程池的使用
date: 2018-8-5
---
## 线程池的概念

线程池（英语：thread pool）：一种线程使用模式。线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建与销毁线程的代价。

## Dubbo中的线程池

Dubbo是一款高性能的分布式服务框架，它实现了透明化的远程调用，只需要通过配置，就能像调用当前jvm内的方法一样调用另一个JVM中的方法，而无需关心底层通信细节。

我们在使用Dubbo时，如果只是发布一个服务供其他java进程调用，或是调用另一个JVM进程中的方法，一般不需要显式地关注多线程的使用。但是我们的一个程序，可以发布多个接口，可以并发处理多个接口的调用。或是并发地调用多个远程接口。这显然是底层框架帮我们处理了多线程的工作。

### Dubbo中对Netty线程的使用

Dubbo在传输层默认使用的是Netty作为通信框架。Dubbo作为服务提供者时使用Netty建立Tcp服务端，作为服务使用者时也使用Netty建立Tcp客户端。

Netty是基于java的NIO技术并结合线程池的通信框架。以下是Dubbo建立服务端的代码：
```java
public class NettyServer extends AbstractServer implements Server {

    private static final Logger logger = LoggerFactory.getLogger(NettyServer.class);

    private Map<String, Channel> channels; // <ip:port, channel>

    private ServerBootstrap bootstrap;

    private io.netty.channel.Channel channel;

    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;

    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }

    @Override
    protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();

        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }
    // 省略若干代码
}
```

ServerBootstrap是Netty的一个类，用于建立服务端的功能。它使用了两组线程池，一个是父线程池，用于处理接受客户端的连接请求，因为Dubbo协议只需要监听一个端口，所以这个线程池只需要一个线程即可。另一个是子线程池，每当和一个客户端建立连接之后，就会从此线程池中选择一个线程进行IO操作。

Dubbo中建立tcp客户端也使用了Netty，下面是实现：

```java
public class NettyClient extends AbstractClient {

    private static final Logger logger = LoggerFactory.getLogger(NettyClient.class);

    private static final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup(Constants.DEFAULT_IO_THREADS, new DefaultThreadFactory("NettyClientWorker", true));

    private Bootstrap bootstrap;

    private volatile Channel channel; // volatile, please copy reference to use

    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        super(url, wrapChannelHandler(url, handler));
    }

    @Override
    protected void doOpen() throws Throwable {
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        if (getTimeout() < 3000) {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
        } else {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout());
        }

        bootstrap.handler(new ChannelInitializer() {

            @Override
            protected void initChannel(Channel ch) throws Exception {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("handler", nettyClientHandler);
            }
        });
    }
```

Netty客户端只需要一个线程池就可以。

以上是Dubbo中对于IO线程池的使用。此外，Dubbo发送请求和处理请求也使用了线程池。

### Dubbo业务线程池的使用

### Dubbo中线程池的种类

Dubbo中使用的线程池都是基于Java提供的ThreadPoolExecutor类。它基于不同的策略提供了创建不同的ThreadPoolExecutor的线程池工厂。包括以下几种：

<b>CachedThreadPool</b> : 缓存线程池，它创建的是一个可以扩展的线程池，可以配置核心线程数量，最大线程数量，任务队列长度，空闲线程保活时间。这个线程池默认的队列长度是java中int类型的最大值。

<b>EagerThreadPool</b> : 这个线程池工厂提供的是一个Dubbo提供自定义的线程池类EagerThreadPoolExecutor，这个线程池扩展了java中的ThreadPoolExecutor。当所有的核心线程都处于忙碌状态时，若有新任务到来，将会直接创建新线程，而不是放到任务队列中。

<b>FixedThreadPool</b> : 固定线程池，它创建的是一个拥有固定线程数量的线程池。用户可配置相关参数。这个线程池是默认选项。

<b>LimitedThreadPool</b> : 可伸缩线程池。它提供的线程池的特色是，线程池设定的线程保活时间是Long类型的最大值。因此它的线程不会减少，只会增加。

Dubbo中对业务线程池的配置是在下面这个类中实现的。

```java
public class WrappedChannelHandler implements ChannelHandlerDelegate {

    protected static final Logger logger = LoggerFactory.getLogger(WrappedChannelHandler.class);

    protected static final ExecutorService SHARED_EXECUTOR = Executors.newCachedThreadPool(new NamedThreadFactory("DubboSharedHandler", true));

    protected final ExecutorService executor;

    protected final ChannelHandler handler;

    protected final URL url;

    public WrappedChannelHandler(ChannelHandler handler, URL url) {
        this.handler = handler;
        this.url = url;
        executor = (ExecutorService) ExtensionLoader.getExtensionLoader(ThreadPool.class).getAdaptiveExtension().getExecutor(url);

        String componentKey = Constants.EXECUTOR_SERVICE_COMPONENT_KEY;
        if (Constants.CONSUMER_SIDE.equalsIgnoreCase(url.getParameter(Constants.SIDE_KEY))) {
            componentKey = Constants.CONSUMER_SIDE;
        }
        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        dataStore.put(componentKey, Integer.toString(url.getPort()), executor);
    }
```

这个类持有了一个线程池接口ExecutorService executor。当实例化这个类时，会根据URL的参数获取对应的线程池，也就是上面说的四种线程池中的一种。

以上是对Dubbo中线程池使用的简要介绍，还有很多地方没有深入。后续有机会再分析。