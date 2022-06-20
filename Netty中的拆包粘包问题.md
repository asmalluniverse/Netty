### Netty中的拆包粘包

TCP传输协议是基于数据流传输的，而基于流化的数据是没有界限的，当客户端向服务端发送数据时，可能会把一个完整的数据报文拆分成多个小报文进行发送，也可能将多个报文合并成一个大报文进行发送。

可能会存在以下问题：

- 服务端恰巧读到了两个完整的数据包 A 和 B，没有出现拆包/粘包问题；
- 服务端接收到 A 和 B 粘在一起的数据包，服务端需要解析出 A 和 B；
- 服务端收到完整的 A 和 B 的一部分数据包 B-1，服务端需要解析出完整的 A，并等待读取完整的 B
  数据包；
- 服务端接收到 A 的一部分数据包 A-1，此时需要等待接收到完整的 A 数据包；
- 数据包 A 较大，服务端需要多次才可以接收完数据包 A。

由于存在拆包/粘包问题，接收方很难界定数据包的边界在哪里，所以可能会读取到不完整的数据导致数据解析出现问题。

### 如何解决拆包和粘包问题呢？

一般我们会在应用层定义通信协议。其实思想也很简单，就是通信双方约定一个通信报文协议，服务端收到报文之后，按照约定的协议进行解码，从而避免出现粘包和拆包问题。

1. 消息长度固定
2. 特定分隔符
3. 消息长度加消息内容加分隔符

Netty中就帮我们实现了相关的协议，进行解决此问题，即Netty中的编解码器



#### Netty中的编解码器

1. FixedLengthFrameDecoder解码器

固定长度解码器FixedLengthFrameDecoder的原理很简单，就是通过构造方法设置一个固定消息大小frameLength，无论接收方一次收到多大的数据，都会严格按照frameLength进行解码。
如果累计读取的长度大小为frameLength的消息，那么解码器会认为已经获取到了一个完整的消息，如
果消息长度小于frameLength，那么该解码器会一直等待后续数据包的达到，直到获得指定长度后返
回。

2. DelimiterBasedFrameDecoder解码器

特殊分隔符解码器： DelimiterBasedFrameDecoder，它有以下几个属性

- delimiters，delimiters指定特殊分隔符，参数类型是ByteBuf，ByteBuf可以传递一个数组，意味着我们可以同时指定多个分隔符，但最终会选择长度最短的分隔符进行拆分。
  比如接收方收到的消息体为
  `hello\nworld\r\n`
  此时指定多个分隔符\n 和\r\n ，那么最终会选择最短的分隔符解码，得到如下数据
  `hello | world |`
- maxLength，表示报文的最大长度限制，如果超过maxLength还没检测到指定分隔符，将会抛出
  TooLongFrameException。
- failFast，表示容错机制，它与maxLength配合使用。如果failFast=true，当超过maxLength后会
  立刻抛出TooLongFrameException，不再进行解码。如果failFast=false，那么会等到解码出一个
  完整的消息后才会抛出TooLongFrameException
- stripDelimiter，它的作用是判断解码后的消息是否去除分隔符，如果stripDelimiter=false，而
  制定的特定分隔符是\n ，那么数据解码的方式如下。
  `hello\nworld\r\n`
  当stripDelimiter=false时，解码后得到
  `hello\n | world\r\n`

3. LengthFieldBasedFrameDecoder解码器

LengthFieldBasedFrameDecoder是长度域解码器，它是解决拆包粘包最常用的解码器，基本上能覆盖
大部分基于长度拆包的场景。其中开源的消息中间件RocketMQ就是使用该解码器进行解码的。
首先来说明一下该解码器的核心参数

- lengthFieldOffset，长度字段的偏移量，也就是存放长度数据的起始位置
- lengthFieldLength，长度字段锁占用的字节数
- lengthAdjustment，在一些较为复杂的协议设计中，长度域不仅仅包含消息的长度，还包含其他
  数据比如版本号、数据类型、数据状态等，这个时候我们可以使用lengthAdjustment进行修正，
  它的值=包体的长度值-长度域的值
- initialBytesToStrip，解码后需要跳过的初始字节数，也就是消息内容字段的起始位置
- lengthFieldEndOffset，长度字段结束的偏移量， 该属性的值=lengthFieldOffset+lengthFieldLength