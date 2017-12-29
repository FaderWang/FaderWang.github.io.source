---
title: 使用java NIO实现一个echo server
date: 2017-12-19 19:34:13
tags: java
---

*废话不多说，直接上代码*

### 客户端代码

```java
public class Client {

    private Selector selector;

    private SocketChannel socketChannel;

    /**
     * 服务器地址
     */
    private String hostIp;

    /**
     * 监听端口
     */
    private int listenPort;

    public Client(String hostIp, int listenPort) throws IOException {
        this.hostIp = hostIp;
        this.listenPort = listenPort;
        initialize();
    }

    private void initialize() throws IOException {
        //开启监听通道，设置为非阻塞模式
        socketChannel = SocketChannel.open(new InetSocketAddress(hostIp, listenPort));
        socketChannel.configureBlocking(false);

        //打开选择器，并注册通道
        selector = Selector.open();
        socketChannel.register(selector, SelectionKey.OP_READ);

        ExecutorService executorService = Executors.newSingleThreadExecutor();
        executorService.submit(() -> {
            try {
                while (selector.select() > 0) {
                    Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
                    while (keyIterator.hasNext()) {
                        SelectionKey key = keyIterator.next();
                        if (key.isReadable()) {
                            SocketChannel channel = (SocketChannel) key.channel();
                            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                            channel.read(byteBuffer);
                            byteBuffer.flip();

                            String receive = Charset.forName("UTF-8").newDecoder().decode(byteBuffer).toString();
                            System.out.println("接收到来自服务器的消息：" + receive);
                            System.out.println("服务器地址：" + channel.socket().getRemoteSocketAddress());
                            key.interestOps(SelectionKey.OP_READ);
                        }
                        keyIterator.remove();
                    }
                }
            } catch (IOException e) {
				e.printStackTrace();
            }
        });
    }

    public void sendMsg(String message) throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.wrap(message.getBytes("UTF-8"));
        socketChannel.write(byteBuffer);

    }

    public static void main(String[] args) throws IOException {
        Client client = new Client("192.168.5.111", 1978);
        client.sendMsg("你好NIO, I am FaderWang");

    }
```

###### 服务端代码

```java

public class Server {

    //缓冲区大小
    private static final int BUFFER_SIZE = 1024;

    //超时时间
    private static final int TIME_OUT = 3000;

    private ServerSocketChannel serverSocketChannel;

    private Selector selector;

    /**
     * 本地监听端口
     */
    private int listenPort;

    public Server(int listenPort) throws IOException {
        this.listenPort = listenPort;
        initialize();
    }

    public void initialize() throws IOException {
        //打开监听通道并绑定端口
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(listenPort));
        serverSocketChannel.configureBlocking(false);

        //开启选择器并注册通道
        selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        //创建一个处理协议实现类
        TCPHandler tcpHandler = new TCPHandlerImpl(BUFFER_SIZE);
        while (true) {
            if (selector.select(TIME_OUT) == 0) {
                System.out.println("独自等待");
                continue;
            }

            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                try {
                    if (key.isAcceptable()) {
                        tcpHandler.handleAccept(key);
                    }
                    if (key.isReadable()) {
                        tcpHandler.handleRead(key);
                    }
                } catch (Exception e) {
                    keyIterator.remove();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException {
        Server server = new Server(1978);
    }
```



###### 服务端处理类

```java
public interface TCPHandler {

    /**
     * 处理接收
     * @param key
     */
    void handleAccept(SelectionKey key) throws Exception;

    /**
     * 处理写入
     * @param key
     */
    void handleWrite(SelectionKey key);


    /**
     * 处理读出
     * @param key
     */
    void handleRead(SelectionKey key) throws IOException;
}

public class TCPHandlerImpl implements TCPHandler {

    private int bufferSize;

    public TCPHandlerImpl(int bufferSize) {
        this.bufferSize = bufferSize;
    }

    @Override
    public void handleAccept(SelectionKey key) throws Exception {
        SocketChannel socketChannel = ((ServerSocketChannel) key.channel()).accept();
        socketChannel.configureBlocking(false);
        Selector selector = key.selector();
        socketChannel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(bufferSize));
    }

    @Override
    public void handleWrite(SelectionKey key) {
        return;
    }

    @Override
    public void handleRead(SelectionKey key) throws IOException {
        SocketChannel socketChannel = (SocketChannel) key.channel();

        //得到缓冲区
        ByteBuffer byteBuffer = (ByteBuffer) key.attachment();
        byteBuffer.clear();

        if (socketChannel.read(byteBuffer) == -1) {
            socketChannel.close();
        } else {
            byteBuffer.flip();
            String receive = Charset.forName("UTF-8").newDecoder().decode(byteBuffer).toString();
            System.out.println("接收到来自客户端的消息：" + receive);
            System.out.println("客户端地址：" + socketChannel.socket().getRemoteSocketAddress());

            String send = "你好，客户端" + new Date().toString() + ",已收到你的消息";
            byteBuffer = ByteBuffer.wrap(send.getBytes("UTF-8"));
            socketChannel.write(byteBuffer);

            //设置为下一次读取或写入做准备
            key.interestOps(SelectionKey.OP_READ);
        }

    }
}
```

