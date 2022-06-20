### ByteBuf

Netty 中ByteBuff等同于JAVA NIO中的ByteBuffer，但是功能上做了很多增强。

##### ByteBuff创建方式

1. 第一种，创建基于堆内存的ByteBuf

```java
ByteBuf buffer=ByteBufAllocator.DEFAULT.heapBuffer(10);
```



2. 第二种，创建基于直接内存（堆外内存）的ByteBuf（默认情况下用的是这种）

   ```java 
   ByteBufAllocator.DEFAULT.directBuffer(10);
   ```

   

   Java中的内存分为两个部分，一部分是不需要jvm管理的直接内存，也被称为堆外内存。堆
   外内存就是把内存对象分配在JVM堆以外的内存区域，这部分内存不是虚拟机管理，而是由
   操作系统来管理，这样可以减少垃圾回收对应用程序的影响

3. 两者的优缺点

	- 优点

直接内存（堆外内存）的好处是读写性能会高一些，如果数据存放在堆中，此时需要把Java堆空间的数据发送到远程服务器，首先需要把堆内部的数据拷贝到直接内存（堆外内存），然后再发送。如果是把数据直接存储到堆外内存中，发送的时候就少了一个复制步骤。

 - 缺点

直接内存需要自己维护，防止内存溢出，而堆内存有JVM帮我们管理。

##### 区别

1. 是ByteBuf可以自动扩容，默认长度是256，如果内容长度超过阈值时，会自动触发扩容
2. ByteBuf采用了池化技术，核心思想就是对象的复用，减少对象的频繁创建和销毁带来的性能开销，池化功能是否开启，可以通过下面的环境变量来控制，其中unpooled表示不开启。`-Dio.netty.allocator.type={unpooled|pooled}`
3. 提供了更加丰富的api



##### ByteBuf存储结构

ByteBuf的存储结构如图3-1所示，从这个图中可以看到ByteBuf其实是一个字节容器，该容器中包含四个部分

1. 已经丢弃的字节，这部分数据是无效的；
2. 可读字节，这部分数据是ByteBuf的主体数据，从ByteBuf里面读取的数据都来自这部分； 
3. 可写字节，所有写到ByteBuf的数据都会存储到这一段；
4. 可扩容字节，表示ByteBuf最多还能扩容多少容量。

![image-20220405160437776](E:\gupao\netty\图片\bytebuf存储结构.png)

在ByteBuf中，有两个指针

- readerIndex： 读指针，每读取一个字节，readerIndex自增加1。ByteBuf里面总共witeIndex-readerIndex
  个字节可读，当readerIndex和writeIndex相等的时候，ByteBuf不可读
- writeIndex： 写指针，每写入一个字节，writeIndex自增加1，直到增加到capacity后，可以触发
  扩容后继续写入。
- ByteBuf中还有一个maxCapacity最大容量，默认的值是Integer.MAX_VALUE ，当ByteBuf写入数
  据时，如果容量不足时，会触发扩容，直到capacity扩容到maxCapacity。

##### ByteBuf中常用的方法

对于ByteBuf来说，常见的方法就是写入和读取

1. Write相关方法
   对于write方法来说，ByteBuf提供了针对各种不同数据类型的写入，比如

- writeChar，写入char类型
- writeInt，写入int类型
- writeFloat，写入float类型
- writeBytes， 写入nio的ByteBuffer
- writeCharSequence， 写入字符串

2. Reader相关方法
   reader方法也同样针对不同数据类型提供了不同的操作方法

   readByte ，读取单个字节
   readInt ， 读取一个int类型
   readFloat ，读取一个float类型

##### 扩容机制

当向ByteBuf写入数据时，发现容量不足时，会触发扩容，而具体的扩容规则是，假设ByteBuf初始容量是10。

1. 如果写入后数据大小未超过512个字节，则选择下一个16的整数倍进行库容。 比如写入数据后大小为12，则扩容后的capacity是16。
2. 如果写入后数据大小超过512个字节，则选择下一个2n。 比如写入后大小是512字节，则扩容后的capacity是210=1024 。（因为29=512，长度已经不够了）扩容不能超过max capacity，否则会报错。

##### ByteBuf的零拷贝机制

ByteBuf的零拷贝机制和我们之前提到的操作系统层面的零拷贝不同，操作系统层面的零拷贝，是我们要把一个文件发送到远程服务器时，需要从内核空间拷贝到用户空间，再从用户空间拷贝到内核空间的网卡缓冲区发送，导致拷贝次数增加。

而ByteBuf中的零拷贝思想也是相同，都是减少数据复制提升性能。假设有一个原始ByteBuf，我们想对这个ByteBuf其中的两个部分的数据进行操作。按照正常的思路，我们会创建两个新的ByteBuf，然后把原始ByteBuf中的部分数据拷贝到两个新的ByteBuf中，但是这种会涉及到数据拷贝，在并发量较大的情况下，会影响到性能。

- slice

ByteBuf中提供了一个slice方法，这个方法可以在不做数据拷贝的情况下对原始ByteBuf，通过索引下表进行拆分，使用方法如下

```java
public static void main(String[] args) {
    ByteBuf buf= ByteBufAllocator.DEFAULT.buffer();//可自动扩容
    buf.writeBytes(new byte[]{1,2,3,4,5,6,7,8,9,10});
    log(buf);
    ByteBuf bb1=buf.slice(0,5);
    ByteBuf bb2=buf.slice(5,5);
    log(bb1);
    log(bb2);
    System.out.println("修改原始数据");
    buf.setByte(2, 5); //修改原始buf数据
    log(bb1);//再打印bb1的结果，发现数据发生了变化
}
```

为了证明slice是没有数据拷贝，我们通过修改原始buf的索引2所在的值，然后再打印第一个分片bb1，可以发现bb1的结果发生了变化。说明两个分片和原始buf指向的数据是同一个。

##### Unpooled

Unpooled相当于Netty中提供给我们的工具类，它是同了非池化的ByteBuf的创建、组合、复制等操作。

1. CompositeByteBuf

假设有一个协议数据，它有头部和消息体组成，这两个部分分别放在两个ByteBuf中,我们就可以通过compositeByteBuf方法实现零拷贝

```java 
ByteBuf header=...
ByteBuf body= ...
```

```java 
public class ByteBufExample {
    public static void main(String[] args) {
        ByteBuf header= ByteBufAllocator.DEFAULT.buffer();//可自动扩容
        header.writeCharSequence("header", CharsetUtil.UTF_8);
        ByteBuf body=ByteBufAllocator.DEFAULT.buffer();
        body.writeCharSequence("body", CharsetUtil.UTF_8);
        CompositeByteBuf compositeByteBuf=Unpooled.compositeBuffer();
        //其中第一个参数是 true, 表示当添加新的 ByteBuf 时, 自动递增 CompositeByteBuf
        的 writeIndex.
        //默认是false，也就是writeIndex=0，这样的话我们不可能从compositeByteBuf中读取到
        数据。
        compositeByteBuf.addComponents(true,header,body);
        log(compositeByteBuf);
    }
    private static void log(ByteBuf buf){
        StringBuilder builder=new StringBuilder()
        .append(" read index:").append(buf.readerIndex())
        .append(" write index:").append(buf.writerIndex())
        .append(" capacity:").append(buf.capacity())
        .append(StringUtil.NEWLINE);
        //把ByteBuf中的内容，dump到StringBuilder中
        ByteBufUtil.appendPrettyHexDump(builder,buf);
        System.out.println(builder.toString());
    }
}
```

之所以CompositeByteBuf能够实现零拷贝，是因为在组合header和body时，并没有对这两个数据进行复制，而是通过CompositeByteBuf构建了一个逻辑整体，里面仍然是两个真实对象，也就是有一个指针指向了同一个对象，所以这里类似于浅拷贝的实现。

2. copiedBuffer

```java
ByteBuf copiedBuffer = Unpooled.copiedBuffer(header, body);
log(copiedBuffer);
header.setByte(1,9);
log(copiedBuffer);
```

copiedBuffer，和wrappedBuffer最大的区别是，该方法会实现数据复制，，修改了原始ByteBuf的值，并没有影响到copiedBuffer