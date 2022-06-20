1.netty执行流程
	1.Server启动，Netty从BoosGroup中选出一个NioEventLoop对指定的端口进行监听（BoosGroup进行连接处理）
	2.Client启动，Netty从EventLoopGroup中选中一个NioEventLoop连接Server，并处理Server发送来的数据。
	3.Client连接Server的Port创建Channel
	4.Netty从ChildGroup中选出一个NioEventLoop与该channel绑定，用于处理该Channel中后续的io操作。
	5.Client通过Channel向Server端发送数据包
	6.Server端接收Channel中的数据，通过Pipeline进行处理
	7.如果Server端也需要向Client发送数据，则需要经过pipeline中处理器处理成的ByteBuf数据包进行传输，通过channel发送到Client端。

2.Netty中的核心组件
Channel
	Channel是 Java NIO的一个基本构造。可以看作是传入或传出数据的载体。因此，它可以被打开或关闭，连接或者断开连接。

EventLoop 与 EventLoopGroup
EventLoop 定义了Netty的核心抽象，用来处理连接的生命周期中所发生的事件，在内部，将会为每个Channel分配一个EventLoop。
Netty 为每个 Channel 分配了一个 EventLoop，用于处理用户连接请求、对用户请求的处理等所有事件。EventLoop 本身只是一个线程驱动，在其生命周期内只会绑定一个线程，让该线程处理一个 Channel 的所有 IO 事件。
EventLoop与线程的关系是 1:1
EventLoopGroup 是一个 EventLoop 池，包含很多的 EventLoop。

ServerBootstrap 与 Bootstrap
	Bootstrap， 客户端启动api，用来链接远程netty server，只绑定一个EventLoopGroup
	ServerBootStrap，服务端监听api，用来监听指定端口，会绑定两个EventLoopGroup，
	bootstrap组件可以非常方便快捷的启动Netty应用程序
	
ChannelHandler 与 ChannelPipeline
	ChannelHandler 是对 Channel 中数据的处理器，这些处理器可以是系统本身定义好的编解码器，也可以是用户自定义的。这些处理器会被统一添加到一个 ChannelPipeline 的对象中，然后按照添加的顺序对 Channel 中的数据进行依次处理。
	
ChannelFuture
Netty 中所有的 I/O 操作都是异步的，即操作不会立即得到返回结果，所以 Netty 中定义了一个 ChannelFuture 对象作为这个异步操作的“代言人”，表示异步操作本身。如果想获取到该异步操作的返回值，可以通过该异步操作对象的addListener() 方法为该异步操作添加监 NIO 网络编程框架 Netty 听器，为其注册回调：当结果出来后马上调用执行。

Netty 的异步编程模型都是建立在 Future 与回调概念之上的。