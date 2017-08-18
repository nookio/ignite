---
title: Thrift源码分析--Transport
date: 2017-08-16 09:43:35
tags:
  - RPC
  - 原理
  - 源码
  - Thrift
---
## 简介    

Thrift是一个RPC调用框架，因此底层会封装一层传输层，用来帮助构建好的代码进行数据的传输。  
其中TTransport封装了传输层，同时他也封装了上层的流。比如他的一个子类：TIOStreamTransport。里面使用的就是我们常用的socket的InputStream和outPutStream  

<!--more-->  

TTransport的设计理念是和我们生成的代码、协议层完全解耦。
 - 我们生成的代码（Client）只需要处理读到的数据以及处理，并不需要关心如何去读取这个数据。
 - 协议层也只需要进行数据的编解码。但是无需关心这些数据是如何来的，是使用的http还是socket还是file等。

### TTransport结构  
![TTransport](/images/TTransport.jpg "TTransport结构")  
这个并不是一个完整的传输层，还有一部分是在服务端使用的，用来帮助生成的代码来创建一个默认的TTransport，供给服务端使用。如果不好理解，可以类比Socket和ServerSocket。  

### TServerTransport结构  
![TServerTransport](/images/TServerTransport.jpg "TServerTransport结构")  


## 分析  

### TTransport  
下面是源码分析，已经对注解翻译，并且去掉了具体实现  

```java  
public abstract class TTransport {
  // 判断传输是否打开，
  public abstract boolean isOpen();

  // 判断是否还有新的数据来
  public boolean peek() { return isOpen(); }

  // 打开传输层，可以用来读写数据了
  public abstract void open() throws TTransportException;

  // 关闭
  public abstract void close();

  // 读取指定长度的数据
  public abstract int read(byte[] buf, int off, int len) throws TTransportException;

  // 把数据全部读取出来
  public int readAll(byte[] buf, int off, int len) throws TTransportException;

  // 写数据，实际调用的是下面的方法
  public void write(byte[] buf) throws TTransportException;

  // 写数据
  public abstract void write(byte[] buf, int off, int len) throws TTransportException;

  // 把缓冲区的数据全部都push出去
  public void flush() throws TTransportException;
}
```

__在实现类中，有这么几个比较重要的子类：__  
 - __TIOStreamTransport:__ 这个类封装了InputStream和OutputStream这两个流，用来处理数据传输中的输入输出流。采用的是阻塞同步IO。  
   - __TSocket:__ 是上面这个类的子类，并且封装了Socket接口。  
 - __TNonblockingTransport:__ 这个类是非阻塞IO的抽象类。
   - __TNonblockingSocket:__ 则是使用了SocketChannel进行了非阻塞IO。
 - __TFileTransport:__ 这个类没有仔细研究，里面允许client把文件传输给服务端，同时允许服务端把文件写入到文件。
 - __TFramedTransport:__ 帧传输类就是按照一帧的固定大小来传输数据，所有的写操作首先都是在内存中完成的直到调用了flush操作，
然后传输节点在flush操作之后将所有数据根据数据的有效载荷写入数据的长度的二进制块发送出去，允许在接收的另一端按照固定的长度来读取。我司的封装是这里的cv操作。
 - __TFastFramedTransport:__ 是快读类，相对于上面的类读取的效率会变的更高。  

__下面从源码角度分析几个比较重要的类:__  

*CASE：TIOStreamTransport，主要以翻译、删减无用代码尽可能突出主干*

```java  
public class TIOStreamTransport extends TTransport {

  // Underlying inputStream
  protected InputStream inputStream_ = null;

  // Underlying outputStream
  protected OutputStream outputStream_ = null;

  // 这里一共有四个构造方法，主要是对内部的两个传输流进行赋值
  protected TIOStreamTransport() {}

  // 传入传输流在构造的时候就已经完成了打开，因此时时都是打开的
  public boolean isOpen() {
    return true;
  }

  // 直接抛异常，两个流必须在构造的时候就已经打开了
  public void open() throws TTransportException {}

  // 关闭流，调用两个流的close方法。去掉了其中的异常处理
  public void close() {
    inputStream_.close();
    inputStream_ = null;
    outputStream_.close();
    outputStream_ = null;
  }

  // 读取数据，调用inputstream中的read方法
  public int read(byte[] buf, int off, int len) throws TTransportException {
    int bytesRead = inputStream_.read(buf, off, len);
    return bytesRead;
  }

  // 写数据，调用outputstream中的read方法
  public void write(byte[] buf, int off, int len) throws TTransportException {
      outputStream_.write(buf, off, len);
  }

  // 把缓冲区的数据push掉
  public void flush() throws TTransportException {
      outputStream_.flush();
  }
}  
```

*下面是他的唯一子类，TSocket，封装了Socket*

```java  
public class TSocket extends TIOStreamTransport {

  // socket
  private Socket socket_ = null;

  // 远程的地址
  private String host_  = null;

  // 远程端口
  private int port_ = 0;

  // 超时时间
  private int timeout_ = 0;

  // 构造方法，使用已经建立链接的socket进行操作
  public TSocket(Socket socket) throws TTransportException {
    socket_ = socket;
    if (isOpen()) {
       inputStream_ = new BufferedInputStream(socket_.getInputStream(), 1024);
       outputStream_ = new BufferedOutputStream(socket_.getOutputStream(), 1024);
    }
  }

  // 构造方法
  public TSocket(String host, int port, int timeout) {
    // 省略直接赋值操作
    initSocket();
  }

  // 创建新的链接
  private void initSocket() {
    socket_ = new Socket();
    // 省略掉赋值操作
  }

  // 判断是否链接上了
  public boolean isOpen() {
    return socket_.isConnected();
  }

  // 打开链接
  public void open() throws TTransportException {
     socket_.connect(new InetSocketAddress(host_, port_), timeout_);
     inputStream_ = new BufferedInputStream(socket_.getInputStream(), 1024);
     outputStream_ = new BufferedOutputStream(socket_.getOutputStream(), 1024);
  }

  // 关闭链接
  public void close() {
    // 关闭stream的关闭方法
    super.close();
    socket_.close();
    socket_ = null;
  }

}
```

*CASE:TNonblockingTransport抽象类 主要以翻译为主*  

```java
public abstract class TNonblockingTransport extends TTransport {

  // 详情可以看下SocketChannel的connect方法，开启链接
  public abstract boolean startConnect() throws IOException;

  // 详情可以看下SocketChannel的finishConnect方法，关闭链接
  public abstract boolean finishConnect() throws IOException;

  // 注册到远程的selector
  public abstract SelectionKey registerSelector(Selector selector, int interests) throws IOException;

  // 读取数据，采用了ByteBuffer这个缓冲区
  public abstract int read(ByteBuffer buffer) throws IOException;


  // 写入数据，采用了ByteBuffer这个缓冲区
  public abstract int write(ByteBuffer buffer) throws IOException;
}
```

*下面介绍的是他的唯一子类：TNonblockingSocket，需要看到懒加载的位置在哪里，暂时未知，为什么不需要flush呢*  
```java
public class TNonblockingSocket extends TNonblockingTransport {
  /**
   * Host and port if passed in, used for lazy non-blocking connect.
   */
  private final SocketAddress socketAddress_;

  private final SocketChannel socketChannel_;

  //省略了若干了构造方法，核心构造方法
  private TNonblockingSocket(SocketChannel socketChannel, int timeout, SocketAddress socketAddress) throws IOException {
    socketChannel_ = socketChannel;
    socketAddress_ = socketAddress;
    // 设置非阻塞信道
    socketChannel.configureBlocking(false);

    // 设置该信道里面的socket参数
    Socket socket = socketChannel.socket();
    socket.setSoLinger(false, 0);
    socket.setTcpNoDelay(true);
    setTimeout(timeout);
  }

  // 注册一个新的选择器
  public SelectionKey registerSelector(Selector selector, int interests) throws IOException {
    return socketChannel_.register(selector, interests);
  }

  // 设置超时时间
  public void setTimeout(int timeout) {
     socketChannel_.socket().setSoTimeout(timeout);
  }

  // 判断当前信道是否开启
  public boolean isOpen() {
    // isConnect方法并不会在关闭以后返回false，所以这里使用isOpen方法
    return socketChannel_.isOpen() && socketChannel_.isConnected();
  }

  // 实现类实现了懒加载，所以不需要手动打开
  public void open() throws TTransportException {
    throw new RuntimeException("open() is not implemented for TNonblockingSocket");
  }

  // 使用ByteBuffer缓冲区读数据
  public int read(ByteBuffer buffer) throws IOException {
    return socketChannel_.read(buffer);
  }

  // 使用ByteBuffer缓冲区写数据
  public int write(ByteBuffer buffer) throws IOException {
    return socketChannel_.write(buffer);
  }

  // 不支持flush，为什么呢。没有缓冲区么
  public void flush() throws TTransportException {
    // Not supported by SocketChannel.
  }

  // 关闭链接
  public void close() {
     socketChannel_.close();
  }

  /** {@inheritDoc} */
  public boolean startConnect() throws IOException {
    return socketChannel_.connect(socketAddress_);
  }

  /** {@inheritDoc} */
  public boolean finishConnect() throws IOException {
    return socketChannel_.finishConnect();
  }

}
```
