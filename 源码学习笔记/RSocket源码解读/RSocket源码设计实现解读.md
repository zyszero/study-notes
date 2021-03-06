# RSocket源码设计实现解读

## 主线

版本以 rsocket 1.6.0.RC版本为准

代码还在不断迭代，现在是1.7.0.RC 了，所以目前看源码，不需要看太仔细，看个主要流程即可。

```java
RSocket协议 对消息包装  => frame => Payload，有了消息之后，就要有服务器 => ServerTransport => 有了服务器，要有数据交流，交流需要凭借 => socket => NIO channle => Netty channel => Reactor Netty 的 Connection => RSocket DuplexConnection
 
RSocketFactory => ClientRSocketFactory
RSocketFactory => ServerRSocketFactory
RSocket  => ResponderRSocket（服务端） => RSocketResponder
RSocket => RSocketRequester（客户端）
对接 Reactor Netty ，对Connection 的加强
connection.send   装配access 
connection.receive  通过access 下发 
RScoketRequester：
如何send元素？
ServerTransport.ServerStart
DuplexConnectionClient
Spring Message 
Message{ Payload } => GenericMessage
MessageChannel => 拓展的几个类
MessageHeaders
```



Netty 如何与 Reactor Netty 接轨？

Reactor Netty 如何与 RSocket 接轨？



RSocket 如何与 Spring Message 接轨？

暂时先不管

RSocket 的生命周期，主线是什么？



RSocket 是基于响应式流（Reactive Stream）语义的应用消息协议。

RSocket协议 =》 即约定好结构的消息体 =》 frame => Payload

类似：

```java
class Frame{
header
data
tail
}
```

有了消息的定义，服务端需要有消息传输器  io.rsocket.transport.ServerTransport：

```java
/** A server contract for writing transports of RSocket. */
public interface ServerTransport<T extends Closeable> extends Transport {

  /**
   * Starts this server.
   *
   * @param acceptor An acceptor to process a newly accepted {@code DuplexConnection}
   * @param mtu The mtu used for fragmentation - if set to zero fragmentation will be disabled
   * @return A handle to retrieve information about a started server.
   * @throws NullPointerException if {@code acceptor} is {@code null}
   */
  Mono<T> start(ConnectionAcceptor acceptor, int mtu);

  /** A contract to accept a new {@code DuplexConnection}. */
  interface ConnectionAcceptor extends Function<DuplexConnection, Publisher<Void>> {

    /**
     * Accept a new {@code DuplexConnection} and returns {@code Publisher} signifying the end of
     * processing of the connection.
     *
     * @param duplexConnection New {@code DuplexConnection} to be processed.
     * @return A {@code Publisher} which terminates when the processing of the connection finishes.
     */
    @Override
    Mono<Void> apply(DuplexConnection duplexConnection);
  }
}
```

消息需要进行send或receive => io.rsocket.DuplexConnection

```java
/** Represents a connection with input/output that the protocol uses. */
public interface DuplexConnection extends Availability, Closeable {

  /**
   * Sends the source of Frames on this connection and returns the {@code Publisher} representing
   * the result of this send.
   *
   * <p><strong>Flow control</strong>
   *
   * <p>The passed {@code Publisher} must
   *
   * @param frames Stream of {@code Frame}s to send on the connection.
   * @return {@code Publisher} that completes when all the frames are written on the connection
   *     successfully and errors when it fails.
   * @throws NullPointerException if {@code frames} is {@code null}
   */
  Mono<Void> send(Publisher<ByteBuf> frames);

  /**
   * Sends a single {@code Frame} on this connection and returns the {@code Publisher} representing
   * the result of this send.
   *
   * @param frame {@code Frame} to send.
   * @return {@code Publisher} that completes when the frame is written on the connection
   *     successfully and errors when it fails.
   */
  default Mono<Void> sendOne(ByteBuf frame) {
    return send(Mono.just(frame));
  }

  /**
   * Returns a stream of all {@code Frame}s received on this connection.
   *
   * <p><strong>Completion</strong>
   *
   * <p>Returned {@code Publisher} <em>MUST</em> never emit a completion event ({@link
   * Subscriber#onComplete()}.
   *
   * <p><strong>Error</strong>
   *
   * <p>Returned {@code Publisher} can error with various transport errors. If the underlying
   * physical connection is closed by the peer, then the returned stream from here <em>MUST</em>
   * emit an {@link ClosedChannelException}.
   *
   * <p><strong>Multiple Subscriptions</strong>
   *
   * <p>Returned {@code Publisher} is not required to support multiple concurrent subscriptions.
   * RSocket will never have multiple subscriptions to this source. Implementations <em>MUST</em>
   * emit an {@link IllegalStateException} for subsequent concurrent subscriptions, if they do not
   * support multiple concurrent subscriptions.
   *
   * @return Stream of all {@code Frame}s received.
   */
  Flux<ByteBuf> receive();

  @Override
  default double availability() {
    return isDisposed() ? 0.0 : 1.0;
  }
}
```

消息传输需要依托于服务器： org.springframework.boot.rsocket.netty.NettyRSocketServer（与 springboot 接轨）

org.springframework.boot.rsocket.netty.NettyRSocketServerFactory#create

```java
@Override
public NettyRSocketServer create(SocketAcceptor socketAcceptor) {
	ServerTransport<CloseableChannel> transport = createTransport();
	RSocketFactory.ServerRSocketFactory factory = RSocketFactory.receive();
	for (ServerRSocketFactoryProcessor processor : this.socketFactoryProcessors) {
		factory = processor.process(factory);
	}
	Mono<CloseableChannel> starter = factory.acceptor(socketAcceptor).transport(transport).start();
	return new NettyRSocketServer(starter, this.lifecycleTimeout);
}

private ServerTransport<CloseableChannel> createTransport() {
	if (this.transport == RSocketServer.Transport.WEBSOCKET) {
		return createWebSocketTransport();
	}
	return createTcpTransport();
}
```



这里以 io.rsocket.transport.ServerTransport 为例：

```java
Mono<CloseableChannel> starter = factory.acceptor(socketAcceptor).transport(transport).start();
```



相当于：io.rsocket.RSocketFactory.ServerRSocketFactory.ServerStart.transport(transport).start();

```java
private class ServerStart<T extends Closeable> implements Start<T>, ServerTransportAcceptor {
	private Supplier<ServerTransport<T>> transportServer;
    ...
    @Override
    public Mono<T> start() {
      return Mono.defer(
          new Supplier<Mono<T>>() {
            ServerSetup serverSetup = serverSetup();
            @Override
            public Mono<T> get() {
              return transportServer
                  .get()
                  .start(duplexConnection -> acceptor(serverSetup, duplexConnection), mtu)
                  .doOnNext(c -> c.onClose().doFinally(v -> serverSetup.dispose()).subscribe());
            }
          });
    }
    ...
    private Mono<Void> acceptor(ServerSetup serverSetup, DuplexConnection connection) 
    {
      ClientServerInputMultiplexer multiplexer =
          new ClientServerInputMultiplexer(connection, plugins, false);
      return multiplexer
          .asSetupConnection()
          .receive()
          .next()
          .flatMap(startFrame -> accept(serverSetup, startFrame, multiplexer));
    }
    ....
    private Mono<Void> accept(
        ServerSetup serverSetup, ByteBuf startFrame, ClientServerInputMultiplexer multiplexer) {
      switch (FrameHeaderFlyweight.frameType(startFrame)) {
        case SETUP:
          return acceptSetup(serverSetup, startFrame, multiplexer);
        case RESUME:
          return acceptResume(serverSetup, startFrame, multiplexer);
        default:
          return acceptUnknown(startFrame, multiplexer);
      }
    }
    ...
    private Mono<Void> acceptSetup(
      ServerSetup serverSetup, ByteBuf setupFrame, ClientServerInputMultiplexer multiplexer) {
	....
    return serverSetup.acceptRSocketSetup(
        setupFrame,
        multiplexer,
        (keepAliveHandler, wrappedMultiplexer) -> {
          ConnectionSetupPayload setupPayload = ConnectionSetupPayload.create(setupFrame);
          Leases<?> leases = leasesSupplier.get();
          RequesterLeaseHandler requesterLeaseHandler =
              isLeaseEnabled
                  ? new RequesterLeaseHandler.Impl(SERVER_TAG, leases.receiver())
                  : RequesterLeaseHandler.None;
          // 核心理解 rSocketRequester  
          RSocket rSocketRequester =
              new RSocketRequester(
                  allocator,
                  wrappedMultiplexer.asServerConnection(),
                  payloadDecoder,
                  errorConsumer,
                  StreamIdSupplier.serverSupplier(),
                  setupPayload.keepAliveInterval(),
                  setupPayload.keepAliveMaxLifetime(),
                  keepAliveHandler,
                  requesterLeaseHandler);
          if (multiSubscriberRequester) {
            rSocketRequester = new MultiSubscriberRSocket(rSocketRequester);
          }
          RSocket wrappedRSocketRequester = plugins.applyRequester(rSocketRequester);
          return plugins
              .applySocketAcceptorInterceptor(acceptor)
              .accept(setupPayload, wrappedRSocketRequester)
              .onErrorResume(
                  err -> sendError(multiplexer, rejectedSetupError(err)).then(Mono.error(err)))
              .doOnNext(
                  rSocketHandler -> {
                    RSocket wrappedRSocketHandler = plugins.applyResponder(rSocketHandler);
                    ResponderLeaseHandler responderLeaseHandler =
                        isLeaseEnabled
                            ? new ResponderLeaseHandler.Impl<>(
                                SERVER_TAG,
                                allocator,
                                leases.sender(),
                                errorConsumer,
                                leases.stats())
                            : ResponderLeaseHandler.None;
                    // 核心理解这一块 rSocketResponder
                    RSocket rSocketResponder =
                        new RSocketResponder(
                            allocator,
                            wrappedMultiplexer.asClientConnection(),
                            wrappedRSocketHandler,
                            payloadDecoder,
                            errorConsumer,
                            responderLeaseHandler);
                  })
              .doFinally(signalType -> setupPayload.release())
              .then();
        });
  	}    
}
```



服务器有了之后，要有实现 RSocket 接口的类，用于实现以下功能：

RSocket is a binary protocol for use on byte stream transports such as TCP, WebSockets, and Aeron.

It enables the following symmetric interaction models via async message passing over a single connection:

- request/response (stream of 1)
- request/stream (finite stream of many)
- fire-and-forget (no response)
- event subscription (infinite stream of many)

RSocket  => ResponderRSocket（服务端） => RSocketResponder
RSocket => RSocketRequester（客户端）

重点理解 io.rsocket.RSocketResponder



有了io.rsocket.transport.ServerTransport，服务器核心在于启动，启动之后，需要有发送/接收消息的介质，即 io.rsocket.transport.ServerTransport.ConnectionAcceptor：

```java
/** A server contract for writing transports of RSocket. */
public interface ServerTransport<T extends Closeable> extends Transport {
  ...
  
  /** A contract to accept a new {@code DuplexConnection}. */
  interface ConnectionAcceptor extends Function<DuplexConnection, Publisher<Void>> {

    /**
     * Accept a new {@code DuplexConnection} and returns {@code Publisher} signifying the end of
     * processing of the connection.
     *
     * @param duplexConnection New {@code DuplexConnection} to be processed.
     * @return A {@code Publisher} which terminates when the processing of the connection finishes.
     */
    @Override
    Mono<Void> apply(DuplexConnection duplexConnection);
  }
}
```

ConnectionAcceptor 基于 DuplexConnection 来发送消息或接受消息。

DuplexConnection 最底层是基于Socket来做，Socket => NIO Channel =》 Netty Channel =》 Reactor Netty Connection => RSocket DuplexConnection

有了DuplexConnection 之后，那么常规意义上的request、response 在RSocket 中，体现为：

io.rsocket.core.RSocketRequester

io.rsocket.ResponderRSocket

都基于io.rsocket.RSocket，体现核心交互模型

```java
/**
 * A contract providing different interaction models for <a
 * href="https://github.com/RSocket/reactivesocket/blob/master/Protocol.md">RSocket protocol</a>.
 */
public interface RSocket extends Availability, Closeable {

  /**
   * Fire and Forget interaction model of {@code RSocket}.
   *
   * @param payload Request payload.
   * @return {@code Publisher} that completes when the passed {@code payload} is successfully
   *     handled, otherwise errors.
   */
  Mono<Void> fireAndForget(Payload payload);

  /**
   * Request-Response interaction model of {@code RSocket}.
   *
   * @param payload Request payload.
   * @return {@code Publisher} containing at most a single {@code Payload} representing the
   *     response.
   */
  Mono<Payload> requestResponse(Payload payload);

  /**
   * Request-Stream interaction model of {@code RSocket}.
   *
   * @param payload Request payload.
   * @return {@code Publisher} containing the stream of {@code Payload}s representing the response.
   */
  Flux<Payload> requestStream(Payload payload);

  /**
   * Request-Channel interaction model of {@code RSocket}.
   *
   * @param payloads Stream of request payloads.
   * @return Stream of response payloads.
   */
  Flux<Payload> requestChannel(Publisher<Payload> payloads);

  /**
   * Metadata-Push interaction model of {@code RSocket}.
   *
   * @param payload Request payloads.
   * @return {@code Publisher} that completes when the passed {@code payload} is successfully
   *     handled, otherwise errors.
   */
  Mono<Void> metadataPush(Payload payload);

  @Override
  default double availability() {
    return isDisposed() ? 0.0 : 1.0;
  }
}
```

回到 io.rsocket.core.RSocketRequester，除了以上重点方法之外，额外需要重点理解的是RSocketRequester如何与Reactor Netty 进行交互，即RSocket 如何与 Reactor Netty 进行对接？

答案就是通过 DuplexConnection 进行对接

涉及到的核心类：

io.rsocket.internal.UnboundedProcessor =》 数据获取源头，相当于Publisher

io.rsocket.lease.ResponderLeaseHandler => 租约（约定，相当于定时器？）

```java
class RSocketResponder implements ResponderRSocket {
  ...  
  RSocketResponder(
      ByteBufAllocator allocator,
      DuplexConnection connection,
      RSocket requestHandler,
      PayloadDecoder payloadDecoder,
      Consumer<Throwable> errorConsumer,
      ResponderLeaseHandler leaseHandler) {
    this.allocator = allocator;
    this.connection = connection;

    this.requestHandler = requestHandler;
    this.responderRSocket =
        (requestHandler instanceof ResponderRSocket) ? (ResponderRSocket) requestHandler : null;

    this.payloadDecoder = payloadDecoder;
    this.errorConsumer = errorConsumer;
    this.leaseHandler = leaseHandler;
    this.sendingLimitableSubscriptions = new SynchronizedIntObjectHashMap<>();
    this.sendingSubscriptions = new SynchronizedIntObjectHashMap<>();
    this.channelProcessors = new SynchronizedIntObjectHashMap<>();

    // DO NOT Change the order here. The Send processor must be subscribed to before receiving
    // connections
    this.sendProcessor = new UnboundedProcessor<>();
	// connection 对接 Reactor Netty的 connection，send(sendProcessor)，相当于配置了Reactor Netty Connection的数据源（Publisher）。
    connection
        .send(sendProcessor)
        .doFinally(this::handleSendProcessorCancel)
        .subscribe(null, this::handleSendProcessorError);
	// 核心关注点，connection.receive() 获取到元素后，通过handleFrame的处理，将元素放到sendProcessor中。
    Disposable receiveDisposable = connection.receive().subscribe(this::handleFrame, errorConsumer);
    Disposable sendLeaseDisposable = leaseHandler.send(sendProcessor::onNext);

    this.connection
        .onClose()
        .doFinally(
            s -> {
              cleanup();
              receiveDisposable.dispose();
              sendLeaseDisposable.dispose();
            })
        .subscribe(null, errorConsumer);
  }	
}
```

为什么 io.rsocket.RSocketResponder#handleFireAndForget 中，的BaseSubscriber 无需事先 onNext 方法？

```java
private void handleFireAndForget(int streamId, Mono<Void> result) {
  result.subscribe(
      new BaseSubscriber<Void>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
          sendingSubscriptions.put(streamId, subscription);
          subscription.request(Long.MAX_VALUE);
        }
        @Override
        protected void hookOnError(Throwable throwable) {
          errorConsumer.accept(throwable);
        }
        @Override
        protected void hookFinally(SignalType type) {
          sendingSubscriptions.remove(streamId);
        }
      });
}
```

因为**fireAndForget**方法是阅后即焚，即只接受不响应，所以无需继续下发元素到reactor netty，即不会使用到 sendProcessor。

对比 **handleRequestResponse** ：

```java
private void handleRequestResponse(int streamId, Mono<Payload> response) {
  response.subscribe(
      new BaseSubscriber<Payload>() {
        ...
        @Override
        protected void hookOnNext(Payload payload) {
          ...
          // 关注点
          sendProcessor.onNext(byteBuf);
        }
        ...
      });
}
```

io.rsocket.core.RSocketRequester 设计思路与 io.rsocket.RSocketResponder的设计思路基本是一致的，可以按照以上思路去理解即可。

DuplexConnection 如何与 reactor netty 进行对接？

以io.rsocket.transport.netty.TcpDuplexConnection 为例：

```java
public final class TcpDuplexConnection extends BaseDuplexConnection {
  private final Connection connection;
  ... 
  @Override
  public Flux<ByteBuf> receive() {
    return connection.inbound().receive().map(this::decode);
  } 
    
  @Override
  public Mono<Void> send(Publisher<ByteBuf> frames) {
    if (frames instanceof Mono) {
      return connection.outbound().sendObject(((Mono<ByteBuf>) frames).map(this::encode)).then();
    }
    return connection.outbound().send(Flux.from(frames).map(this::encode)).then();
  }
  ... 
}
```

这里的Connection为 reactor.netty.channel.ChannelOperations

```java
public class ChannelOperations<INBOUND extends NettyInbound, OUTBOUND extends NettyOutbound>
		implements NettyInbound, NettyOutbound, Connection, CoreSubscriber<Void> {

	@Override
	public ByteBufFlux receive() {
		return ByteBufFlux.fromInbound(receiveObject(), connection.channel()
		                                                          .alloc());
	}    
    
	@Override
	@SuppressWarnings("unchecked")
	public NettyOutbound sendObject(Publisher<?> dataStream, Predicate<Object> predicate) {
		if (!channel().isActive()) {
			return then(Mono.error(new AbortedException("Connection has been closed BEFORE send operation")));
		}
		if (dataStream instanceof Mono) {
			return then(((Mono<?>)dataStream).flatMap(m -> FutureMono.from(channel().writeAndFlush(m)))
			                                 .doOnDiscard(ReferenceCounted.class, ReferenceCounted::release));
		}
		return then(MonoSendMany.objectSource(dataStream, channel(), predicate));
	}    
}
```

设计代码，首要考虑资源释放：

```java
public abstract class BaseDuplexConnection implements DuplexConnection {
  private MonoProcessor<Void> onClose = MonoProcessor.create();

  public BaseDuplexConnection() {
    onClose.doFinally(s -> doOnClose()).subscribe();
  }

  protected abstract void doOnClose();

  @Override
  public final Mono<Void> onClose() {
    return onClose;
  }

  @Override
  public final void dispose() {
    onClose.onComplete();
  }

  @Override
  public final boolean isDisposed() {
    return onClose.isDisposed();
  }
}
```

任务：以官方给出的例子出发，一整套流程进行理解。



## 疑问

reactor.core.publisher.FluxBuffer.BufferSkipSubscriber#scanUnsafe

![](images/rsocket源码疑问-1.png)

其作用与getter方法一致，为什么要这样写？

因为对外要统一API，如果使用getter方法的话，就得明确每个类有哪些属性的getter，这样才能获得值。使用统一使用`scanUnsafe`方法，不仅外部调用方便，而且外部调用者无需关心内部实现机制，只需传key即可，无则返回null。一般用于统计等分析用途。