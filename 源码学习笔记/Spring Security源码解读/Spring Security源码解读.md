# Spring Security 源码解读系列

如何从零构造一个Spring Security框架？（或者构造一个权限框架）

核心是过滤器，设置一个请求处理入口，即“门卫”，不同用户需要应用不同的过滤规则，所以要有个过滤规则管理器，即“门禁系统”，“门卫”管控“门禁系统”   用户对应状态的管理



## Spring Security 4.x

首先看配置，核心配置接口：

```java
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {
	/**
	 * Initialize the {@link SecurityBuilder}. Here only shared state should be created
	 * and modified, but not properties on the {@link SecurityBuilder} used for building
	 * the object. This ensures that the {@link #configure(SecurityBuilder)} method uses
	 * the correct shared objects when building.
	 *
	 * @param builder
	 * @throws Exception
	 */
	void init(B builder) throws Exception;

	/**
	 * Configure the {@link SecurityBuilder} by setting the necessary properties on the
	 * {@link SecurityBuilder}.
	 *
	 * @param builder
	 * @throws Exception
	 */
	void configure(B builder) throws Exception;
}
```

一切配置类，将围绕着这两个核心方法进行拓展



### 如何将 Spring Security 融入到Spring 容器中，并最终汇入Servlet的FilterChain中



### 权限管理设计实现与异常处理

org.springframework.security.config.annotation.web.AbstractRequestMatcherRegistry

```java
	public class RequestMatcherConfigurer
			extends AbstractRequestMatcherRegistry<RequestMatcherConfigurer> {
    	
    }
```





org.springframework.security.web.access.intercept.FilterSecurityInterceptor