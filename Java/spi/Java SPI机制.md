# Java SPI 机制

## 背景

这个技术出现的背景、初衷和要达到什么样的目标或是要解决什么样的问题。

> SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。

![](C:\study\study-notes\Java\spi\images\spi.jpg)

## 优劣势

这个技术的优势和劣势分别是什么，或者说，这个技术的 trade-off 是什么。

优势：

- 解耦：服务提供方做任何修改，都不会影响到服务使用方。
- 可插拔：比如JDBC。

劣势：

- 做不到按需加载，JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
- 如果扩展点加载失败，连扩展点的名称都拿不到了。

## 场景

这个技术适用的场景。

- JDBC
- JNDI
- Java XML Processing API
- Locael
- NIO Channel Provider
- 准备插件化的地方

## 技术的组成部分和关键点

- 核心实现类：ServiceLoader
- 核心要点：
  - 在META-INF/services/目录中创建以Service接口全限定名命名的文件，该文件内容为Service接口具体实现类的全限定名，文件编码必须为UTF-8。
  - 使用ServiceLoader.load(Class class); 动态加载Service接口的实现类。
  - 如SPI的实现类为jar，则需要将其放在当前程序的classpath下。
  - Service的具体实现类必须有一个不带参数的构造方法。

## 实现原理

技术的底层原理和关键实现。

底层原理是：反射。

Java SPI机制的实现原理：

> 外部使用时，往往通过`ServiceLoader#load(Class<S> service, ClassLoader loader)`或`ServiceLoader#load(Class<S> service)`调用，最后都是在`reload`方法中创建了`LazyIterator`对象，`LazyIterator`是`ServiceLoader`的内部类，实现了`Iterator`接口，其作用是一个懒加载的迭代器，在`hasNextService`方法中，完成了对位于`META-INF/services/`目录下的配置文件的解析，并在`nextService`方法中，完成了对具体实现类的实例化。
>
> `META-INF/services/`，是`ServiceLoader`中约定的接口与实现类的关系配置目录，文件名是接口全限定类名，内容是接口对应的具体实现类，如果有多个实现类，分别将不同的实现类都分别作为每一行去配置。解析过程中，通过`LinkedHashMap<String,S>`数据结构的`providers`，将已经发现了的接口实现类进行了缓存，并对外提供的`iterator()`方法，方便外部遍历。

关键实现：

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{
    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }
    
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ...
        return ServiceLoader.load(service, cl);    
    }
    
    public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
    }

    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        ...
        reload();
    }
    
    private class LazyIterator
        implements Iterator<S>
    {
        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                ...
            }
        }

        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                ...
            }
            ...
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
               ...
            }
            ...
        }        
}
```

## Dubbo SPI vs JDK SPI

### Dubbo SPI 简介

> ### [来源：](https://dubbo.apache.org/zh/docs/v2.7/dev/spi/#来源)
>
> Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。
>
> Dubbo 改进了 JDK 标准的 SPI 的以下问题：
>
> - JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> - 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 `getName()` 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。
> - 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。
>
> ### 约定：
>
> 在扩展类的 jar 包内 [1](https://dubbo.apache.org/zh/docs/v2.7/dev/spi/#fn:1)，放置扩展点配置文件 `META-INF/dubbo/接口全限定名`，内容为：`配置名=扩展实现类全限定名`，多个实现类用换行符分隔。
>
> ### 示例：
>
> 以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：`META-INF/dubbo/org.apache.dubbo.rpc.Protocol`，内容为：
>
> ```fallback
> xxx=com.alibaba.xxx.XxxProtocol
> ```
>
> 实现类内容 [2](https://dubbo.apache.org/zh/docs/v2.7/dev/spi/#fn:2)：
>
> ```java
> package com.alibaba.xxx;
>  
> import org.apache.dubbo.rpc.Protocol;
>  
> public class XxxProtocol implements Protocol { 
>     // ...
> }
> ```
>
> ### 配置模块中的配置
>
> Dubbo 配置模块中，扩展点均有对应配置属性或标签，通过配置指定使用哪个扩展实现。比如：
>
> ```xml
> <dubbo:protocol name="xxx" />
> ```



