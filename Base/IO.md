### 概述
JAVA的IO大概可以分为以下几类：
* 磁盘操作：File
* 字节操作：InputStream 和 OutputStream
* 字符操作：Reader 和 Writer
* 对象操作：Serializable
* 网络操作：Socket
* 新的输入输出：NIO
***  
### 磁盘操作
File 类可以用来表示文件和目录信息，但是他不表示文件的内容
从 JAVA 7 开始，可以使用 Paths 和 Files 代替 File

***
### 字节操作
#### 装饰者模式
JAVA IO 使用了装饰者模式来实现，以 InputStream 为例：
* InputStream 是抽象组件
* FileInputStream 是 InputStream 的子类，属于具体组件，提供了字节流的输入操作
* FileInputStream 属于抽象装饰者，装饰者用于装饰组件，为组件提供额外的功能，例如BufferedInputStream 为 InputStream 提供缓存的功能
![avatar](/static/InputStream.png)

### 字符操作
#### 编码和解码
编码就是把字符转成字节，而解码是把字节重新组合成字符
char 占16位，也就是两个字节

#### String 的编码方式
String 可以看成一个字符序列，getByte()默认 utf-8

#### Reader 和 Writer
不管是磁盘还是网络传输，最小的存储单元都是字节，但是程序中操作的通常为字符的形式，因此需要提供对字符的操作方式
* InputStreamReader 实现从字节流解码成字符流
* OutputStreamWriter 实现从字符流编码为字节流

### 对象操作
#### 序列化
将一个对象转成字节序列，方便存储和传输
* 序列化：ObjectOutputStream.writeObject()
* 反序列化：ObjectInputStream.readObject()

不会对静态变量进行序列化，因为序列化只是保存对象的状态，静态变量属于类的状态。

#### Serializable
序列化的类需要实现 Serializable 接口

#### transient
transient 关键字可以使一些属性不会被序列化

### 网络操作
Java 中的网络支持：
* InetAddress：用于标识网络上的硬件资源，即 IP 地址
* URL：统一资源定位符
* Sockets：使用 TCP 协议实现网络通信
* Datagram：使用 UDP 协议实现网络通信

#### InetAddress
#### URL
可以直接从 URL 中读取字节流数据
#### Sockets
* ServerSocket：服务器类
* Socket：客户端类
* 服务器和客户端通过 InputStream 和 OutputStream 进入输入输出
![avatar](../static/Socket.png)

#### Datagram
* DatagramSocket：通信类
* DatagramPacket：数据包类
  
### NIO
新的输入/输出(NIO)库在JDK 1.4 中引入，提供了高速的、面向块的IO
#### 流和块
IO 和 NIO 最重要的区别在于数据打包和传输的方式，IO 以流的方式处理数据，而 NIO 以块的方式处理数据

面向流的IO一次能处理一个字节的数据：一个输入流产生一个字节数据，一个输出流消费一个字节数据。面向流的IO通常相当慢
面向块的 I/O 一次处理一个数据块，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

java.io.* 已经以 NIO 为基础重新实现了，所以现在它可以利用 NIO 的一些特性。例如，java.io.* 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。

#### 通道和缓冲区
##### 1.通道
通道 Channel 是对原IO包中流的模拟，可以通过它读取或写入数据
通道和流不同之处在于，流只能在一个方向上移动（一个流必须是InputStream或OutputStream 的子类），而通道是双向的，可以用于读、写或者读同时写

通道包含以下类型：
* FileChannel：文件中读写
* DatagramChannle：UDP 读写网络数据
* SocketChannel：TCP 读写网络数据
* ServerSocketChannel：可以监听新进来的 TCP 链接，对每一个新进来的链接都会创建一个 SocketChannel

##### 2.缓冲区
发送一个通道的所有数据必须先放到缓冲区，同样的，从通道中读取的任何数据都要先读到缓冲区。也就是说，不会直接对通道进行读写，而是要先经过缓冲区

缓冲区实质上是一个数组，但他不仅仅是一个数组，缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程

缓冲区包含以下类型：
* ByteBuffer
* CharBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* FloatBuffer
* DoubleBuffer

#### 缓冲区状态变量
* capacity：最大容量
* position： 当前已读写的字节数
* limit：还可以读写的字节数

#### 选择器
NIO 常常被叫做非阻塞 IO，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用

NIO 实现了IO 多路复用中的 Reactor 模型，一个线程 Thread 使用一个选择器 Selector 通过轮询方式去监听多个通道 Channel 上的事件，从而让一个线程可以处理多个事件

通过配置监听的通道 Channel 为非阻塞，当 Channel 上的IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其他线程 Channel，找到IO 事件已经达到的 Channel 执行

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件，对于 IO 密集型的应用具有很好地性能。

应该注意的是，只有套接字 Channel 才能配置为非阻塞，而 FileChannel 不能，为 FileChannel 配置非阻塞也没有意义。
  
#### 内存映射文件
内存映射文件 I/O 是一种读和写文件数据的方法，它可以比常规的基于流或者基于通道的 I/O 快得多。

向内存映射文件写入可能是危险的，只是改变数组的单个元素这样的简单操作，就可能会直接修改磁盘上的文件。修改数据与将数据保存到磁盘是没有分开的。

下面代码行将文件的前 1024 个字节映射到内存中，map() 方法返回一个 MappedByteBuffer，它是 ByteBuffer 的子类。因此，可以像使用其他任何 ByteBuffer 一样使用新映射的缓冲区，操作系统会在需要时负责执行映射。

#### 对比
NIO 与普通 I/O 的区别主要有以下两点：

NIO 是非阻塞的；
NIO 面向块，I/O 面向流。
