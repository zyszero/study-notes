# Spring Reactor 全新解读系列 2019-2020 全集 

## 01 响应式标准接口从无到有设计



## 02 响应式背压来源与设计及Flux的Create浅析

响应式的背压机制，可以类比为大坝，具有“蓄洪”的作用。

即可以理解为：“可以存储，元素有顺序” =》 队列，大坝有可能会决堤，所以这个队列是有界的，那么对应也有无界模式。

背压设计位置在上游，即Publisher处。

拿 reactor.core.publisher.FluxCreate 来举例

```java
final class FluxCreate<T> extends Flux<T> implements SourceProducer<T> {
	@Override
	public void subscribe(CoreSubscriber<? super T> actual) {
		BaseSink<T> sink = createSink(actual, backpressure);

		actual.onSubscribe(sink);
		try {
            // 1、
			source.accept(
					createMode == CreateMode.PUSH_PULL ? new SerializedSink<>(sink) :
							sink);
		}
		catch (Throwable ex) {
			Exceptions.throwIfFatal(ex);
			sink.error(Operators.onOperatorError(ex, actual.currentContext()));
		}
	}
}    
```

可以拿饭馆的场景进行类比，饭馆厨师是Publisher，顾客是Subscriber，那么顾客点的单则是Subscription（单纯只是类比），拿以上代码来看的话，BaseSink 是Subscription，另一方面也充当着服务员的职责，负责上菜。

步骤1，则可以看做是厨师做饭的过程，SerializedSink 则可以看成是服务员plus。

reactor.core.publisher.FluxCreate.SerializedSink：

```java
	static final class SerializedSink<T> implements FluxSink<T>, Scannable {
		final BaseSink<T> sink;
        ...
		@Override
		public FluxSink<T> next(T t) {
			Objects.requireNonNull(t, "t is null in sink.next(t)");
			if (sink.isTerminated() || done) {
				Operators.onNextDropped(t, sink.currentContext());
				return this;
			}
			if (WIP.get(this) == 0 && WIP.compareAndSet(this, 0, 1)) {
				try {
					sink.next(t);
				}
				catch (Throwable ex) {
					Operators.onOperatorError(sink, ex, t, sink.currentContext());
				}
				if (WIP.decrementAndGet(this) == 0) {
					return this;
				}
			}
			else {
				this.mpscQueue.offer(t);
				if (WIP.getAndIncrement(this) != 0) {
					return this;
				}
			}
			drainLoop();
			return this;
		}

		void drainLoop() {
			Context ctx = sink.currentContext();
			BaseSink<T> e = sink;
			Queue<T> q = mpscQueue;
			for (; ; ) {

				for (; ; ) {
					if (e.isCancelled()) {
						Operators.onDiscardQueueWithClear(q, ctx, null);
						if (WIP.decrementAndGet(this) == 0) {
							return;
						}
						else {
							continue;
						}
					}

					if (ERROR.get(this) != null) {
						Operators.onDiscardQueueWithClear(q, ctx, null);
						//noinspection ConstantConditions
						e.error(Exceptions.terminate(ERROR, this));
						return;
					}

					boolean d = done;
					T v = q.poll();

					boolean empty = v == null;

					if (d && empty) {
						e.complete();
						return;
					}

					if (empty) {
						break;
					}

					try {
                        // 对接到BaseSink
						e.next(v);
					}
					catch (Throwable ex) {
						Operators.onOperatorError(sink, ex, v, sink.currentContext());
					}
				}

				if (WIP.decrementAndGet(this) == 0) {
					break;
				}
			}
		}        
    }
```

以 reactor.core.publisher.FluxCreate.BufferAsyncSink 为例：

```java
	static final class BufferAsyncSink<T> extends BaseSink<T> {
		// 背压机制的容器
        final Queue<T> queue;
   		....
		@Override
		public FluxSink<T> next(T t) {
			queue.offer(t);
			drain();
			return this;
		}        
		//impl note: don't use isTerminated() in the drain loop,
		//it needs to first check the `done` status before setting `disposable` to TERMINATED
		//otherwise it would either loose the ability to drain or the ability to invoke the
		//handler at the right time.
		void drain() {
			if (WIP.getAndIncrement(this) != 0) {
				return;
			}

			final Subscriber<? super T> a = actual;
			final Queue<T> q = queue;

			for (; ; ) {
				long r = requested;
				long e = 0L;

				while (e != r) {
					if (isCancelled()) {
						Operators.onDiscardQueueWithClear(q, ctx, null);
						if (WIP.decrementAndGet(this) != 0) {
							continue;
						}
						else {
							return;
						}
					}

					boolean d = done;

					T o = q.poll();

					boolean empty = o == null;

					if (d && empty) {
						Throwable ex = error;
						if (ex != null) {
							super.error(ex);
						}
						else {
							super.complete();
						}
						return;
					}

					if (empty) {
						break;
					}
					// 连接到了下游 Subscriber.onNext
					a.onNext(o);

					e++;
				}

				if (e == r) {
					if (isCancelled()) {
						Operators.onDiscardQueueWithClear(q, ctx, null);
						if (WIP.decrementAndGet(this) != 0) {
							continue;
						}
						else {
							return;
						}
					}

					boolean d = done;

					boolean empty = q.isEmpty();

					if (d && empty) {
						Throwable ex = error;
						if (ex != null) {
							super.error(ex);
						}
						else {
							super.complete();
						}
						return;
					}
				}

				if (e != 0) {
					Operators.produced(REQUESTED, this, e);
				}

				if (WIP.decrementAndGet(this) == 0) {
					break;
				}
			}
		}
	}
```

## 03 函数编程的入门与思维方式



## 04 - 06 Reactor中前后一体化套路实现演进

![](images/04-06 Reactor中前后一体化套路实现演进.png)

- 电影类比 Publisher
- 观众类比 Subscriber
- 影院类比 Subscription
- 影院工作人员类比 Sink

![](images/04-06 Reactor中前后一体化套路实现演进-02.png)

通过反射去调用方法与new 对象后在调用方法的区别

- new 会分配内存
- JVM加载class字节文件后，会在每个对应的class字节文件的内存中维护一个方法表，通过反射，根据XXX.class，即可直接从方法表中获取方法，无需分配内存。



比较契合以上套路实现演进的类：reactor.core.publisher.FluxArray

```java
final class FluxArray<T> extends Flux<T> implements Fuseable, SourceProducer<T> {
	// 类比电影
    final T[] array;
    ....
	@SuppressWarnings("unchecked")
	public static <T> void subscribe(CoreSubscriber<? super T> s, T[] array) {
		if (array.length == 0) {
			Operators.complete(s);
			return;
		}
		if (s instanceof ConditionalSubscriber) {
			s.onSubscribe(new ArrayConditionalSubscription<>((ConditionalSubscriber<? super T>) s, array));
		}
		else {
            // 类比顾客前往影院
			s.onSubscribe(new ArraySubscription<>(s, array));
		}
	}

	@Override
	public void subscribe(CoreSubscriber<? super T> actual) {
		subscribe(actual, array);
	}        
}
```

reactor.core.publisher.LambdaMonoSubscriber

```java
final class LambdaMonoSubscriber<T> implements InnerConsumer<T>, Disposable {
	@Override
	public final void onSubscribe(Subscription s) {
		if (Operators.validate(subscription, s)) {
			this.subscription = s;

			if (subscriptionConsumer != null) {
				try {
                    // 类比挑选影院
					subscriptionConsumer.accept(s);
				}
				catch (Throwable t) {
					Exceptions.throwIfFatal(t);
					s.cancel();
					onError(t);
				}
			}
			else {
                // 类比电影院出票
				s.request(Long.MAX_VALUE);
			}

		}
	}    
}
```

reactor.core.publisher.FluxArray.ArraySubscription：

```java
	static final class ArraySubscription<T>
			implements InnerProducer<T>, SynchronousSubscription<T> {
		@Override
		public void request(long n) {
            // 类比出票逻辑
			if (Operators.validate(n)) {
				if (Operators.addCap(REQUESTED, this, n) == 0) {
					if (n == Long.MAX_VALUE) {
						fastPath();
					}
					else {
						slowPath(n);
					}
				}
			}
		}	
    }
```



## 07 设计模式相关理解

![](images/07 设计模式相关理解.png)



## 08 Spring Reactor 中间操作设计套路解读

![](images/08 - Spring Reactor中间操作设计套路解读.png)

套娃设计

```java
Flux f = flux.map(...)
			 .flatmap(...)
			 .fliter(...)
```

详细设计参考：reactor.core.publisher.InternalFluxOperator

搞懂上述代码的套娃设计是怎么实现的



## 10 链式编程 + 设计模式编程

![](images/10 链式编程 + 设计模式编程.png)



## 11 响应式编程过程逻辑调用梳理总结

Publisher => 工厂

Subscriber => 消费者

Subscription => 产品（可能没有十分准确）

Sink => 经销商

工厂直接对接消费者：Subscriber.onNext()

工厂对接经销商，经销商对接消费者：Sink.next()

![](images/11 响应式编程过程逻辑调用梳理总结.png)

## 12 数理思维结合场景设计实现 Flux.Create

工厂直接对接消费者：

```java
g(x){
    ...
    subscriber.onNext();
    ...
    subscriber.onComplete();    
    ...    
}
```

简单案例：

```java
public abstract class Flux<T> implements CorePublisher<T> {
	public static <T> Flux<T> just(T data) {
		return onAssembly(new FluxJust<>(data));
	}    
}

final class FluxJust<T> extends Flux<T>
		implements Fuseable.ScalarCallable<T>, Fuseable,
		           SourceProducer<T> {
	final T value;

	FluxJust(T value) {
		this.value = Objects.requireNonNull(value, "value");
	}

	@Override
	public void subscribe(final CoreSubscriber<? super T> actual) {
		actual.onSubscribe(new WeakScalarSubscription<>(value, actual));
	}

	static final class WeakScalarSubscription<T> implements QueueSubscription<T>,
	                                                        InnerProducer<T>{
		boolean terminado;
		final T                     value;
		final CoreSubscriber<? super T> actual;

		WeakScalarSubscription(@Nullable T value, CoreSubscriber<? super T> actual) {
			this.value = value;
			this.actual = actual;
		}

		@Override
		public void request(long elements) {
			if (terminado) {
				return;
			}

			terminado = true;
			if (value != null) {
                // Subscriber.onNext
				actual.onNext(value);
			}
			actual.onComplete();
		}                                                                
    }
}

```

复杂案例：reactor.core.publisher.FluxArray

```java
public abstract class Flux<T> implements CorePublisher<T> {
	public static <T> Flux<T> fromArray(T[] array) {
		if (array.length == 0) {
			return empty();
		}
		if (array.length == 1) {
			return just(array[0]);
		}
		return onAssembly(new FluxArray<>(array));
	} 
}

final class FluxArray<T> extends Flux<T> implements Fuseable, SourceProducer<T> {

	final T[] array;

	@SafeVarargs
	public FluxArray(T... array) {
		this.array = Objects.requireNonNull(array, "array");
	}

	@SuppressWarnings("unchecked")
	public static <T> void subscribe(CoreSubscriber<? super T> s, T[] array) {
		...
		if (s instanceof ConditionalSubscriber) {
			s.onSubscribe(new ArrayConditionalSubscription<>((ConditionalSubscriber<? super T>) s, array));
		}
		else {
			s.onSubscribe(new ArraySubscription<>(s, array));
		}
	}

	static final class ArrayConditionalSubscription<T>
			implements InnerProducer<T>, SynchronousSubscription<T> {
		final ConditionalSubscriber<? super T> actual;

		final T[] array;

		int index;

		ArrayConditionalSubscription(ConditionalSubscriber<? super T> actual, T[] array) {
			this.actual = actual;
			this.array = array;
		}

		@Override
		public CoreSubscriber<? super T> actual() {
			return actual;
		}

		@Override
		public void request(long n) {
			if (Operators.validate(n)) {
				if (Operators.addCap(REQUESTED, this, n) == 0) {
					if (n == Long.MAX_VALUE) {
						fastPath();
					}
					else {
						slowPath(n);
					}
				}
			}
		}
        
		void slowPath(long n) {
			final T[] a = array;
			final int len = a.length;
			final ConditionalSubscriber<? super T> s = actual;

			int i = index;
			int e = 0;

			for (; ; ) {
				...
				while (i != len && e != n) {
					T t = a[i];
					...
					// Subscriber.onNext
					boolean b = s.tryOnNext(t);
					...
				}

				if (i == len) {
					s.onComplete();
					return;
				}
				...
			}
		}
        
		void fastPath() {
			final T[] a = array;
			final int len = a.length;
			final Subscriber<? super T> s = actual;

			for (int i = index; i != len; i++) {
				...
				// Subscriber.onNext
				s.onNext(t);
			}
			...
			s.onComplete();
		}    
    }    
}
```

工厂 =》 经销商 =》 消费者：

什么时候使用Sink？或者说为什么会有Sink出现？

因为不同消费者对产品的需求是不一样的。比如个体消费者和企业（集体）消费者，个体对产品的需求数量较少，消费频率较低，由此我们可以发现，不同的消费者，需要有不同的应对策略，而这个应对策略就放在经销商上。

那么工厂就不需要根据消费者具体需求进行定制化了，实现解耦。消费者与经销商进行绑定，各自实现自己的需求（比如个人消费者到商店买到商品，企业消费者到批发经销商处购买商品）

```
g(x){
	...
	sink.next(x){
		... // 控制策略，控制下发速度，数量等等
		Subscriber.onNext()	
	}
	...
	sink.complete(x){
		...
		Subscriber.onComplete()
	}
}
g(x) = g(f(x))
f(x) => Sink
x => Subscriber
```

实际案例：

```java
public abstract class Flux<T> implements CorePublisher<T> {
    
    public static <T> Flux<T> create(Consumer<? super FluxSink<T>> emitter) {
	    return create(emitter, OverflowStrategy.BUFFER);
    }	
    
    public static <T> Flux<T> create(Consumer<? super FluxSink<T>> emitter, OverflowStrategy backpressure) {
		return onAssembly(new FluxCreate<>(emitter, backpressure, FluxCreate.CreateMode.PUSH_PULL));
	}
}

final class FluxCreate<T> extends Flux<T> implements SourceProducer<T> {
    
	@Override
	public void subscribe(CoreSubscriber<? super T> actual) {
		BaseSink<T> sink = createSink(actual, backpressure);

		actual.onSubscribe(sink);
		try {
			source.accept(
                	// SerializedSink 防止不同线程间对数组的并发操作（CAS）
					createMode == CreateMode.PUSH_PULL ? new SerializedSink<>(sink) :
							sink);
		}
		catch (Throwable ex) {
			Exceptions.throwIfFatal(ex);
			sink.error(Operators.onOperatorError(ex, actual.currentContext()));
		}
	}
}
```



