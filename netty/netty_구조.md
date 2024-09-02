# Netty (작성 중)

- EventLoop 기반의 비동기 network-io 애플리케이션 프레임워크
- EventLoop를 통해 감지된 Event를 처리하는 handler가 pipeline을 이루어 각 이벤트를 처리
- ChannelPipeline.java
``` java
 /*                                                 I/O Request
 *                                            via {@link Channel} or
 *                                        {@link ChannelHandlerContext}
 *                                                      |
 *  +---------------------------------------------------+---------------+
 *  |                           ChannelPipeline         |               |
 *  |                                                  \|/              |
 *  |    +---------------------+            +-----------+----------+    |
 *  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  .               |
 *  |               .                                   .               |
 *  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
 *  |        [ method call]                       [method call]         |
 *  |               .                                   .               |
 *  |               .                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  +---------------+-----------------------------------+---------------+
 *                  |                                  \|/
 *  +---------------+-----------------------------------+---------------+
 *  |               |                                   |               |
 *  |       [ Socket.read() ]                    [ Socket.write() ]     |
 *  |                                                                   |
 *  |  Netty Internal I/O Threads (Transport Implementation)            |
 *  +-------------------------------------------------------------------+
 */
```

## 주요 Component

### NioEventLoop
- Nio의 Selector를 이용하여 eventLoop를 구현
- 등록된 channel의 event를 확인하고 Pipline에 event를 발행하여 hanlder들을 실행

### NioEventLoopGroup
- EventLoop의 묵음
- BossGroup과 ChildGroup으로 구분됨
- #### BossGroup
    - `OP_ACCEPT` 이밴트를 담당
    - `ServerSocketChannel`에서 accept 이벤트를 받아 처리
- #### ChildGroup
    - `OP_READ`, `OP_WRITE` 이벤트를 담당
    - `SocketChannel`에서 read, write 이벤트를 받아 처리

### NioServerSocketChannel
- Nio의 `ServerSocketChannel`의 역할
- accept 이벤트 발생
``` java
/**
 * A {@link io.netty.channel.socket.ServerSocketChannel} implementation which uses
 * NIO selector based implementation to accept new connections.
 */
public class NioServerSocketChannel extends AbstractNioMessageChannel { 
    ...
    public NioServerSocketChannel(ServerSocketChannel channel) {
        // OP_ACCEPT만 등록
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
}
```
### NioSocketChannel
- Nio의 `SocketChannel`의 역할
- read, write 이벤트 발생

### ChannelHandlerContext
- 내부에 이벤트를 처리하는 `ChannelHandler`를 가짐
- linkedList 형태로 다음 context와 이전 context를 가지고 있음
- channelInboundInvoker, ChannelOutboundInvoker의 구현체로 이벤트가 발행되면 handler에서 event에 해당하는 메소드를 호출
### ChannelPipeline
- headContext와 tailContext를 가짐
- channelInboundInvoker, ChannelOutboundInvoker의 구현체로 이벤트가 발행되면 headContext에 이벤드를 전달

### 기본적인 Netty 사용 코드
``` java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();

try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup) // 1
            .channel(NioServerSocketChannel.class) // 2
            .handler(new ChannelInitializer<NioServerSocketChannel>() { // 3
                @Override
                protected void initChannel(NioServerSocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    pipeline.addLast(new EchoServerInboundHandler());
                }
            })
            .childHandler(new ChannelInitializer<SocketChannel>() { // 4
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    pipeline.addLast(new EchoServerChildInboundHandler());
                }
            });

    // 서버 시작
    log.info("start server...");
    ChannelFuture f = b.bind(8080).sync(); // 5
    f.channel().closeFuture().sync();
} finally {
    log.info("close server...");
    workerGroup.shutdownGracefully();
    bossGroup.shutdownGracefully();
}
```
1. bossGroup과 childGroup을 설정
2. 비동기 네트워크에 사용될 서버 채널 설정
3. bossGroup 이벤트루프 채널 초기화 설정
4. childGroup 이벤트루프 채널 초기화 설정
5. 포트에 바인딩 후 OP_ACCEPT 이벤트 리스닝 시작

## Netty

### ServerBootstrap.bind()
- bind 함수를 통해 OP_ACCEPT 이벤트가 발생하는 NioServerSocketChannel이 생성된다.
- 생성된 channel은 초기화(init) 과정을 통해 channel의 pipeline에 handler들이 등록된다.
- 초기화를 끝낸 channel은 bossGroup eventLoop에 등록(register)되어 OP_ACCEPT 이벤트를 받기 시작한다.

``` java
public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
    }

private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
     ...
}

final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // channel 생성
        channel = channelFactory.newChannel();
        // channel 초기화
        init(channel);
    } catch (Throwable t) { ... }

    // bossgroup에 channel을 등록
    ChannelFuture regFuture = config().group().register(channel);
    ...
    return regFuture;
}
```

### ServerBootstrap.init()
- bossGroup에 등록될 ServerSocketChannel의 pipeline에 handler를 등록
- 어떤 handler들이 등록될까?
    1. 개발자가 ServerBootstrap.handler를 통해 등록한 handler들을 등록
    2. ServerBootstrapAcceptor 라는 handler를 마지막에 등록

### ServerBootstrapAcceptor
- accept 이벤트를 담당하는 ServerSocketChannel의 pipeline 마지막에 등록된 handler
- childGroup에 등록될 SocketChannel의 Pipeline을 초기화하고, SocketChannel을 ChildGroup의 eventLoop에 등록(register)하는 역할을 함

<p/>
ServerBootstrapAcceptor.channelRead()

``` java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    // SocketChannel에 개발자가 설정한 childHandler를 등록
    child.pipeline().addLast(childHandler);

    ...

    try {
        // SocketChannel을 childGroup에 등록
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) { ... }
}
```

### ServerBootstrapAcceptor의 channelRead 이벤트는 어떻게 발생될까?
NioEventLoop Class
``` java
protected void run() {
    int selectCnt = 0;
    // 무한 루프를 돌며 준비된 channel을 select
    for (;;) {
        try {
            try {
                switch (strategy) {
                    case SelectStrategy.SELECT:
                        try {
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally { ... }
                    default:
                }
            } catch (IOException e) {...}

            ...

            processSelectedKeys()
        }
    }
}

private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}

private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        // selectedKeys를 돌며 processSelectedKey 호출
        final SelectionKey k = selectedKeys.keys[i];
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            processSelectedKey(k, task);
        }
        ...
    }
}

private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();

    try {
        int readyOps = k.readyOps();

        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { ... }
        if ((readyOps & SelectionKey.OP_WRITE) != 0) { ... }
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // OP_READ or OP_ACCEPT시 이벤트가 발생한 channel.unsafe.read 호출
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) { ... }
}
```

- 결과적으로 OP_ACCEPT 이벤트가 발생한 NioServerSocketChannel의 unsafe.read()가 호출
<p/>
AbstractNioMessageChannel.NioMessageUnsafe.read()

``` java
public void read() {
    final ChannelPipeline pipeline = pipeline();
    ...
    try {
        try {
            do {
                int localRead = doReadMessages(readBuf);
                ...
            } while (continueReading(allocHandle));
        } catch (Throwable t) { ... }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            /* 
            * ServerSocketChannel pipeline에 channelRead 이벤트 발행 
            *  -> ServerSocketChannel pipeline 마지막에 등록된 ServerBootstrapAcceptor.channelRead 함수 호출
            */
            pipeline.fireChannelRead(readBuf.get(i));
        }
        ...
    } finally { ... }
}
```