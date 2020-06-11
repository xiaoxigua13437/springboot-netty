# 前言

作者 yushu 

这篇文章主要介绍springBoot 整合 Netty,由于本人也处在学习Netty过程中,并没将Netty运用到实际的项目经验中 ,
这里也是网上搜寻了一些Netty例子学习后总结的,借鉴了一些他人的写法和经验。如有重复部分,还请见谅。

关于springBoot如何整合Netty,将分为以下几步进行分析与讨论:

* 构建netty服务端
* 构建netty客户端
* 利用protobuf定义消息格式
* 服务端空闲检测
* 客户端发送心跳包与断线重连

ps 这里为了简单起见(主要是有点懒,哈哈哈哈......),将Netty客户端与服务端放在了同一个springBoot工程里,当然也可以
将客户端与服务端拆开。

###### _1._ 构建netty服务端
   Netty服务端构建,代码如下 :
   ```java
    @Slf4j
    @Component
    public class NettyServer {
    
        //boss 线程组用于处理连接工作
        private EventLoopGroup boss = new NioEventLoopGroup();
    
        //work 线程组用于数据处理
        private EventLoopGroup work = new NioEventLoopGroup();
    
        @Value(value = "${netty.port}")
        private Integer port;
    
    
        /**
         * 启动Netty Server
         * @throws InterruptedException
         */
        public void start() throws InterruptedException {
    
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(boss,work)
                    //指定channel
                    .channel(NioServerSocketChannel.class)
    
                    //使用指定的端口设置套接字地址
                    .localAddress(new InetSocketAddress(port))
    
                    //服务端可连接队列数,对应TCP/IP协议listen函数中backlog参数
                    .option(ChannelOption.SO_BACKLOG,1024)
    
                    //设置TCP长连接,一般如果两个小时内没有数据的通信时,TCP会自动发送一个活动探测数据报文
                    .childOption(ChannelOption.SO_KEEPALIVE,true)
    
                    //将小的数据包包装成更大的帧进行传送，提高网络的负载
                    .childOption(ChannelOption.TCP_NODELAY,true)
    
                    .childHandler(new NettyServerHandlerInitializer());
            ChannelFuture future = bootstrap.bind().sync();
            if (future.isSuccess()){
                log.info("启动 Netty Server");
    
            }
        }
    
    
        /**
         * 关闭Netty
         * @throws InterruptedException
         */
        @PreDestroy
        public void destory() throws InterruptedException {
            boss.shutdownGracefully().sync();
            work.shutdownGracefully().sync();
            log.info("关闭Netty");
        }
    
    
    
    
    }
    
   ```



###### _2._ 构建netty客户端
   Netty 客户端代码与服务端类似，代码如下：
   ```java
@Slf4j
@Component
public class NettyClient {


    private EventLoopGroup group = new NioEventLoopGroup();

    @Value(value = "${netty.port}")
    private int port;

    @Value(value = "${netty.host}")
    private String host;

    private SocketChannel socketChannel;


    public void sendMsg(MessageBase.Message message){
        socketChannel.writeAndFlush(message);
    }



    public void start(){

        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioServerSocketChannel.class)
                .remoteAddress(host,port)
                .option(ChannelOption.SO_KEEPALIVE,true)
                .option(ChannelOption.TCP_NODELAY,true)
                .handler(new NettyCilentHandlerInitializer());

        ChannelFuture future = bootstrap.connect();
        //客户端断线重连逻辑
        future.addListener((ChannelFutureListener) future1 -> {
            if (future1.isSuccess()) {
                log.info("连接Netty服务端成功");
            } else {
                log.info("连接失败，进行断线重连");
                future1.channel().eventLoop().schedule(() -> start(), 20, TimeUnit.SECONDS);
            }
        });
        socketChannel = (SocketChannel) future.channel();
    }

    
}

   ```

###### _3._ 利用protobuf定义消息格式

在整合使用 Netty 的过程中，我们使用 Google 的protobuf定义消息格式，下面来简单介绍下 protobuf














