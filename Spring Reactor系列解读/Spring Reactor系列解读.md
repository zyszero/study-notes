# Spring Reactor 解读


## 2. 响应式背压来源与设计及Flux的Create浅析
### 2.1 响应式背压来源与设计
背压机制：back pressure，类似于水库大坝，具有“蓄水、控制上游水量，泄洪”等作用。

背压设计的位置 -> 上游 =》 Publisher

## 2.2 Flux的Create浅析

面馆理论

## 3. 
```java
Publisher.subscribe(Subscriber s)
-> Subscriber.onSubscribe(Subscription s)
 -> Subscription.request()
  -> Sink.next():Subscriber.onNext() -> Sink.complete():Subscriber.onComplete()
  -> Subscriber.onNext() -> Subscriber.onComplete()
```

​		



Publisher{subscriber();}

Subscriber{onSubscribe(); onNext(); onError(); onComplete();}

Subscription{request(); cancel();}

Sink(next(); error(); complete();)