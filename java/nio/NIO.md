# Server主流程
```
Selector selector = Selector.open(); // NioEventLoop.openSelector()
ServerSocketChannel serverSocket = ServerSocketChannel.open(); // 对应NioServerSocketChannel.newSocket
serverSocket.bind(new InetSocketAddress("localhost", 5454)); // NioServerSocketChannel.doBind
serverSocket.configureBlocking(false); // AbstractNioChannel()
serverSocket.register(selector, SelectionKey.OP_ACCEPT); // AbstractNioChannel.doRegister()，但是此时并没有注册OP_ACCEPT，
            // 而是到doBeginRead才真正注册进去，在channelActive后被调用
while (true) { // NioEventLoop.run
    selector.select(); // NioEventLoop.select
    Set<SelectionKey> selectedKeys = selector.selectedKeys(); // 这里对SelectionKey的处理稍微有点不太一样，使用了SelectedSelectionKeySet
    Iterator<SelectionKey> iter = selectedKeys.iterator();
    while (iter.hasNext()) { // NioEventLoop.processSelectedKeys -> processSelectedKeysOptimized
        SelectionKey key = iter.next();
 
        if (key.isAcceptable()) { // AbstractNioMessageChannel.read
            register(selector, serverSocket); // NioServerSocketChannel.doReadMessages
        }
 
        if (key.isReadable()) {
            answerWithEcho(buffer, key);
        }
        // this is necessary
        iter.remove();
    }
}


private static void register(Selector selector, ServerSocketChannel serverSocket) throws IOException {
    SocketChannel client = serverSocket.accept(); // NioServerSocketChannel.doReadMessages
    client.configureBlocking(false); // AbstractNioChannel()
    client.register(selector, SelectionKey.OP_READ); // AbstractNioChannel.doBeginRead
}
```

# Client主流程
```
SocketChannel channel = SocketChannel.open() // NioSocketChannel.newSocket
channel.connect(new InetSocketAddress("localhost", 5454)) // NioSocketChannel.doConnect
ByteBuffer buffer = ByteBuffer.allocate(256);

// NioEventLoop.processSelectedKey
client.write(buffer); 
client.read(buffer); // NioSocketChannel.doReadBytes

```

# remark
`SelectedSelectionKeySet`实现了Set接口，通过反射直接设置到`Selector`的`publicKeys`和`publicSelectedKeys`中。底层使用一个数组来存储，每次有新的key就加到结尾，复杂度为O(1)。在reset的时候会清空数组。为了配合这个`SelectedSelectionKeySet`还专门实现了一个`SelectedSelectionKeySetSelector`，里面持有一个正常的`Selector`的引用。