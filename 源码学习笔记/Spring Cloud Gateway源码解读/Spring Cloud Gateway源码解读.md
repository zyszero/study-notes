# Spring Cloud Gateway 源码解读

深入学习Spring Cloud Gateway需要掌握的知识：

![image-20201115223715790](C:\study\study-notes\源码学习笔记\Spring Cloud Gateway源码解读\images\01.png)

## RouterFunction

在使用Spring WebFlux时，除了以往那种通过@RequestMapping进行路径匹配的方式外（即Spring MVC的那一套），还有一种Router的形式，即类似如下代码的配置：

```java
  @Bean
  public RouterFunction<ServerResponse> route(UserHandler userHandler) {

    return RouterFunctions.route(
            RequestPredicates.GET("/users")
                .and(RequestPredicates.accept(MediaType.APPLICATION_JSON_UTF8)),
            userHandler::getAllUsers)
        .andRoute(
            RequestPredicates.GET("/users/{userId}")
                .and(RequestPredicates.accept(MediaType.APPLICATION_JSON_UTF8)),
            userHandler::getUser)
        .andRoute(
            RequestPredicates.POST("/users")
                .and(RequestPredicates.contentType(MediaType.APPLICATION_JSON_UTF8)),
            userHandler::createUser)
        .andRoute(
            RequestPredicates.DELETE("/users/{userId}")
                .and(RequestPredicates.accept(MediaType.APPLICATION_JSON_UTF8)),
            userHandler::deleteUser);
  }
```

那么这种形式的配置，是如何实现类似@RequestMapping的功能的呢？这种配置是如何接入Spring WebFlux的呢？即如何生效的？



RouterFunctions#route(RequestPredicate, HandlerFunction<T>)

```java
// org.springframework.web.reactive.function.server.RouterFunctions
public static <T extends ServerResponse> RouterFunction<T> route(
		RequestPredicate predicate, HandlerFunction<T> handlerFunction) {
	return new DefaultRouterFunction<>(predicate, handlerFunction);
}
```

```java
// org.springframework.web.reactive.function.server.RouterFunctions.DefaultRouterFunction
private static final class DefaultRouterFunction<T extends ServerResponse> extends AbstractRouterFunction<T> {
	public DefaultRouterFunction(RequestPredicate predicate, HandlerFunction<T> handlerFunction) {
	Assert.notNull(predicate, "Predicate must not be null");
	Assert.notNull(handlerFunction, "HandlerFunction must not be null");
	this.predicate = predicate;
	this.handlerFunction = handlerFunction;
}
    @Override
    public Mono<HandlerFunction<T>> route(ServerRequest request) {
        // 路径判断
        if (this.predicate.test(request)) {
            if (logger.isTraceEnabled()) {
                String logPrefix = request.exchange().getLogPrefix();
                logger.trace(logPrefix + String.format("Matched %s", this.predicate));
            }
            return Mono.just(this.handlerFunction);
        }
        else {
            return Mono.empty();
        }
    }    
}
```

```java
@FunctionalInterface
public interface HandlerFunction<T extends ServerResponse> {

	/**
	 * Handle the given request.
	 * @param request the request to handle
	 * @return the response
	 */
	Mono<T> handle(ServerRequest request);

}

public class UserHandler {
  ...	
  public Mono<ServerResponse> getAllUsers(ServerRequest request) {
    return ok().contentType(MediaType.APPLICATION_JSON_UTF8)
        .body(userService.findAll().map(userResourceAssembler::toResource), UserResource.class);
  }

  public Mono<ServerResponse> getUser(ServerRequest request) {
    return userService
        .findById(UUID.fromString(request.pathVariable("userId")))
        .map(userResourceAssembler::toResource)
        .flatMap(
            ur ->
                ok().contentType(MediaType.APPLICATION_JSON_UTF8)
                    .body(BodyInserters.fromObject(ur)))
        .switchIfEmpty(notFound().build());
  }

  public Mono<ServerResponse> deleteUser(ServerRequest request) {
    return ok().build(userService.deleteById(UUID.fromString(request.pathVariable("userId"))));
  }

  public Mono<ServerResponse> createUser(ServerRequest request) {
    return ok().build(
            userService.create(
  request.bodyToMono(CreateUserResource.class).map(userResourceAssembler::toModel)));
  }
}
```



## HandlerMapping

![image-20201115224031069](C:\study\study-notes\源码学习笔记\Spring Cloud Gateway源码解读\images\02.png)



实现了InitializingBean，首先看：afterPropertiesSet方法，类初始化后会执行该方法，在这个方法里，进行RouterFuntion的配置

```java

public class RouterFunctionMapping extends AbstractHandlerMapping implements InitializingBean {
	@Override
	public void afterPropertiesSet() throws Exception {
		if (CollectionUtils.isEmpty(this.messageReaders)) {
			ServerCodecConfigurer codecConfigurer = ServerCodecConfigurer.create();
			this.messageReaders = codecConfigurer.getReaders();
		}

		if (this.routerFunction == null) {
			initRouterFunctions();
		}
	}
    
	/**
	 * Initialized the router functions by detecting them in the application context.
	 */
	protected void initRouterFunctions() {
		List<RouterFunction<?>> routerFunctions = routerFunctions();
		this.routerFunction = routerFunctions.stream().reduce(RouterFunction::andOther).orElse(null);
		logRouterFunctions(routerFunctions);
	}

	private List<RouterFunction<?>> routerFunctions() {
		List<RouterFunction<?>> functions = obtainApplicationContext()
            	// 获取实现了接口RouterFunction的Bean，这里的代码很有借鉴价值
				.getBeanProvider(RouterFunction.class)
				.orderedStream()
				.map(router -> (RouterFunction<?>)router)
				.collect(Collectors.toList());
		return (!CollectionUtils.isEmpty(functions) ? functions : Collections.emptyList());
	}    
}
```



那么RouterFunctionMapping在哪进行配置？

```java
/**
 * {@link EnableAutoConfiguration Auto-configuration} for {@link EnableWebFlux WebFlux}.
 *
 * @author Brian Clozel
 * @author Rob Winch
 * @author Stephane Nicoll
 * @author Andy Wilkinson
 * @author Phillip Webb
 * @author Eddú Meléndez
 * @author Artsiom Yudovin
 * @since 2.0.0
 */
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.REACTIVE)
@ConditionalOnClass(WebFluxConfigurer.class)
@ConditionalOnMissingBean({ WebFluxConfigurationSupport.class })
@AutoConfigureAfter({ ReactiveWebServerFactoryAutoConfiguration.class, CodecsAutoConfiguration.class,
		ValidationAutoConfiguration.class })
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebFluxAutoConfiguration {
    ...
	/**
	 * Configuration equivalent to {@code @EnableWebFlux}.
	 */
	@Configuration(proxyBeanMethods = false)
	public static class EnableWebFluxConfiguration extends DelegatingWebFluxConfiguration {	
        ...
    }
}

/**
 * A subclass of {@code WebFluxConfigurationSupport} that detects and delegates
 * to all beans of type {@link WebFluxConfigurer} allowing them to customize the
 * configuration provided by {@code WebFluxConfigurationSupport}. This is the
 * class actually imported by {@link EnableWebFlux @EnableWebFlux}.
 *
 * @author Brian Clozel
 * @since 5.0
 */
@Configuration(proxyBeanMethods = false)
public class DelegatingWebFluxConfiguration extends WebFluxConfigurationSupport {
    ...
}

public class WebFluxConfigurationSupport implements ApplicationContextAware {
	@Bean
	public RouterFunctionMapping routerFunctionMapping(ServerCodecConfigurer serverCodecConfigurer) {
		RouterFunctionMapping mapping = createRouterFunctionMapping();
		mapping.setOrder(-1);  // go before RequestMappingHandlerMapping
		mapping.setMessageReaders(serverCodecConfigurer.getReaders());
		mapping.setCorsConfigurations(getCorsConfigurations());

		return mapping;
	}    
}
```

由此可知，当使用@EnableWebFlux注解时，就会将RouterFunctionMapping加入到Spring 容器中



## DispatcherHandler

```java
public class DispatcherHandler implements WebHandler, ApplicationContextAware {
	@Nullable
	private List<HandlerMapping> handlerMappings;
    ...
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) {
		initStrategies(applicationContext);
	}


	protected void initStrategies(ApplicationContext context) {
        // 加载容器中所有的HandlerMapping，即包括RouterFunctionMapping
		Map<String, HandlerMapping> mappingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerMapping.class, true, false);

		ArrayList<HandlerMapping> mappings = new ArrayList<>(mappingBeans.values());
		AnnotationAwareOrderComparator.sort(mappings);
		this.handlerMappings = Collections.unmodifiableList(mappings);

		Map<String, HandlerAdapter> adapterBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerAdapter.class, true, false);

		this.handlerAdapters = new ArrayList<>(adapterBeans.values());
		AnnotationAwareOrderComparator.sort(this.handlerAdapters);

		Map<String, HandlerResultHandler> beans = BeanFactoryUtils.beansOfTypeIncludingAncestors(
				context, HandlerResultHandler.class, true, false);

		this.resultHandlers = new ArrayList<>(beans.values());
		AnnotationAwareOrderComparator.sort(this.resultHandlers);
	}
    
	...
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		if (this.handlerMappings == null) {
			return createNotFoundError();
		}
		return Flux.fromIterable(this.handlerMappings)
          		// 对接路由处理
				.concatMap(mapping -> mapping.getHandler(exchange))
				.next()
				.switchIfEmpty(createNotFoundError())
				.flatMap(handler -> invokeHandler(exchange, handler))
				.flatMap(result -> handleResult(exchange, result));
	}
    ...
}
```



那么DispatcherHandler配置在哪呢？

1. 加入到Spring 容器中

```java
public class WebFluxConfigurationSupport implements ApplicationContextAware {
	@Bean
	public DispatcherHandler webHandler() {
		return new DispatcherHandler();
	}    
}
```

2. DispatcherHandler实现了WebHandler接口



## HttpWebHandlerAdapter

```java
public class HttpWebHandlerAdapter extends WebHandlerDecorator implements HttpHandler {
    ...
    // 这里注入DispatcherHandler
	public HttpWebHandlerAdapter(WebHandler delegate) {
		super(delegate);
	}
    
	@Override
	public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
		if (this.forwardedHeaderTransformer != null) {
			request = this.forwardedHeaderTransformer.apply(request);
		}
		ServerWebExchange exchange = createExchange(request, response);

		LogFormatUtils.traceDebug(logger, traceOn ->
				exchange.getLogPrefix() + formatRequest(exchange.getRequest()) +
						(traceOn ? ", headers=" + formatHeaders(exchange.getRequest().getHeaders()) : ""));

		return getDelegate().handle(exchange)
				.doOnSuccess(aVoid -> logResponse(exchange))
				.onErrorResume(ex -> handleUnresolvedError(exchange, ex))
				.then(Mono.defer(response::setComplete));
	}

	protected ServerWebExchange createExchange(ServerHttpRequest request, ServerHttpResponse response) {
		return new DefaultServerWebExchange(request, response, this.sessionManager,
				getCodecConfigurer(), getLocaleContextResolver(), this.applicationContext);
	}
}

public class WebHandlerDecorator implements WebHandler {
	private final WebHandler delegate;
    ... 
	/**
	 * Return the wrapped delegate.
	 */
	public WebHandler getDelegate() {
		return this.delegate;
	}        
}
```



 ReactorHttpHandlerAdapter 用于对接Reactor Netty

```java
public class ReactorHttpHandlerAdapter implements BiFunction<HttpServerRequest, HttpServerResponse, Mono<Void>> {
	private final HttpHandler httpHandler;

	@Override
	public Mono<Void> apply(HttpServerRequest reactorRequest, HttpServerResponse reactorResponse) {
		NettyDataBufferFactory bufferFactory = new NettyDataBufferFactory(reactorResponse.alloc());
		try {
			ReactorServerHttpRequest request = new ReactorServerHttpRequest(reactorRequest, bufferFactory);
			ServerHttpResponse response = new ReactorServerHttpResponse(reactorResponse, bufferFactory);

			if (request.getMethod() == HttpMethod.HEAD) {
				response = new HttpHeadResponseDecorator(response);
			}

			return this.httpHandler.handle(request, response)
					.doOnError(ex -> logger.trace(request.getLogPrefix() + "Failed to complete: " + ex.getMessage()))
					.doOnSuccess(aVoid -> logger.trace(request.getLogPrefix() + "Handling completed"));
		}
		catch (URISyntaxException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to get request URI: " + ex.getMessage());
			}
			reactorResponse.status(HttpResponseStatus.BAD_REQUEST);
			return Mono.empty();
		}
	}    
}
```



## WebHandler

```java
/**
 * Contract to handle a web request.
 *
 * <p>Use {@link HttpWebHandlerAdapter} to adapt a {@code WebHandler} to an
 * {@link org.springframework.http.server.reactive.HttpHandler HttpHandler}.
 * The {@link WebHttpHandlerBuilder} provides a convenient way to do that while
 * also optionally configuring one or more filters and/or exception handlers.
 *
 * @author Rossen Stoyanchev
 * @since 5.0
 */
public interface WebHandler {

	/**
	 * Handle the web server exchange.
	 * @param exchange the current server exchange
	 * @return {@code Mono<Void>} to indicate when request handling is complete
	 */
	Mono<Void> handle(ServerWebExchange exchange);

}
```

![image-20201115234112294](C:\study\study-notes\源码学习笔记\Spring Cloud Gateway源码解读\images\03.png)

如何将DispatcherHandler、HttpWebHandlerAdapter、FilteringWebHandler、ExceptionHandlingWebHandler糅合在一起？

```java
public final class WebHttpHandlerBuilder {
	private final WebHandler webHandler;
    ...
	public static WebHttpHandlerBuilder applicationContext(ApplicationContext context) {
        // String WEB_HANDLER_BEAN_NAME = "webHandler"，往回看，你会发现DispatcherHandler的Bean name为 "webHandler"，即在这里接入DispatcherHandler
		WebHttpHandlerBuilder builder = new WebHttpHandlerBuilder(
				context.getBean(WEB_HANDLER_BEAN_NAME, WebHandler.class), context);
		List<WebFilter> webFilters = context
				.getBeanProvider(WebFilter.class)
				.orderedStream()
				.collect(Collectors.toList());
		builder.filters(filters -> filters.addAll(webFilters));
		List<WebExceptionHandler> exceptionHandlers = context
				.getBeanProvider(WebExceptionHandler.class)
				.orderedStream()
				.collect(Collectors.toList());
		builder.exceptionHandlers(handlers -> handlers.addAll(exceptionHandlers));
		...
	}
    ...    
	/**
	 * Build the {@link HttpHandler}.
	 */
	public HttpHandler build() {
        // this.webHandler是DispatcherHandler
		// 糅合DispatcherHandler、HttpWebHandlerAdapter、FilteringWebHandler和ExceptionHandlingWebHandler
		WebHandler decorated = new FilteringWebHandler(this.webHandler, this.filters);
		decorated = new ExceptionHandlingWebHandler(decorated,  this.exceptionHandlers);
		
		HttpWebHandlerAdapter adapted = new HttpWebHandlerAdapter(decorated);
		if (this.sessionManager != null) {
			adapted.setSessionManager(this.sessionManager);
		}
		if (this.codecConfigurer != null) {
			adapted.setCodecConfigurer(this.codecConfigurer);
		}
		if (this.localeContextResolver != null) {
			adapted.setLocaleContextResolver(this.localeContextResolver);
		}
		if (this.forwardedHeaderTransformer != null) {
			adapted.setForwardedHeaderTransformer(this.forwardedHeaderTransformer);
		}
		if (this.applicationContext != null) {
			adapted.setApplicationContext(this.applicationContext);
		}
		adapted.afterPropertiesSet();

		return adapted;
	}    
}
```

那么ReactorHttpHandlerAdapter#httpHandler，就可以是HttpWebHandlerAdapter

至此，UserHandler -> HandlerFuntion -> RouterFunction（通过RouterFunctions.route方法进行生成，例如DefaultRouterFunction等 ） ->  HandlerMapping -> RouterFunctionMapping -> WebFluxConfigurationSupport（@EnableWebFlux）->  DispatcherHandler#initStrategies -> DispatcherHandler#handle（对接上路由处理方法）



## RouteLocator

Spring Cloud Gateway 中的RouteLocator是什么呢？它又是如何接入Spring WebFlux的呢?

RouteLocator类似上文说到的RouterFunction，配置代码如下：

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
	return builder.routes()
			.route("resource", r -> r.path("/resource")
				.filters(f -> f.filters(filterFactory.apply())
								.removeRequestHeader("Cookie")) // Prevents cookie being sent downstream
				.uri("http://resource:9000")) // Taking advantage of docker naming
			.build();
}
```

RouteLocator有点类似RouterFunction

RouteLocator如何接入到Spring WebFlux，并从中起到作用呢？

与RouterFunction类似，RouterFunction是通过RouterFunctionMapping接入，RouteLocator是通过RoutePredicateHandlerMapping接入的。

```java
public class GatewayAutoConfiguration {
    
    @Bean
	@Primary
	// TODO: property to disable composite?
    // 因为是根据类型自动注入的，所以这里会注入自定义的RouteLocator
	public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
		return new CachingRouteLocator(
				new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
	}
    
    
    // 配置RoutePredicateHandlerMapping，加入到Spring容器中
	@Bean
	public RoutePredicateHandlerMapping routePredicateHandlerMapping(
			FilteringWebHandler webHandler, RouteLocator routeLocator,
			GlobalCorsProperties globalCorsProperties, Environment environment) {
        // 由于cachedCompositeRouteLocator方法上有@Primary，所以这里接入的是cachedCompositeRouteLocator方法返回的RouteLocator
		return new RoutePredicateHandlerMapping(webHandler, routeLocator,
				globalCorsProperties, environment);
	}    
}
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {
    ...
	public RoutePredicateHandlerMapping(FilteringWebHandler webHandler,
			RouteLocator routeLocator, GlobalCorsProperties globalCorsProperties,
			Environment environment) {
		this.webHandler = webHandler;
		this.routeLocator = routeLocator;

		this.managementPort = getPortProperty(environment, "management.server.");
		this.managementPortType = getManagementPortType(environment);
		setOrder(1);
		setCorsConfigurations(globalCorsProperties.getCorsConfigurations());
	}
    ...
}
```

Spring Cloud Gateway 中的 RoutePredicateHandlerMapping也如RouterHandlerMapping一样，将在DispatcherHandler中接入，因为RoutePredicateHandlerMapping 实现了 HandlerMapping接口

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {
	...
}
public abstract class AbstractHandlerMapping extends ApplicationObjectSupport
		implements HandlerMapping, Ordered, BeanNameAware {
	...
}
```

至此RouteLocator加入到了RoutePredicateHandlerMapping，RoutePredicateHandlerMapping接入DispatcherHandler，DispatcherHandler-》HttpWebHandlerAdapter（WebHandler）-》ReactorHttpHandlerAdapter

那么RouteLocator又是如何起作用的呢？

主要看RoutePredicateHandlerMapping的核心方法getHandlerInternal：

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {
	private final FilteringWebHandler webHandler;
    ...
	@Override
	protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		// don't handle requests on management port if set and different than server port
		if (this.managementPortType == DIFFERENT && this.managementPort != null
				&& exchange.getRequest().getURI().getPort() == this.managementPort) {
			return Mono.empty();
		}
		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug(
								"Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}
					// 将router挂载到exchange中
					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for ["
								+ getExchangeDesc(exchange) + "]");
					}
				})));
	}

	protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
        // 拿到自行配置的routeLocator，然后进行处理
		return this.routeLocator.getRoutes()
				// individually filter routes so that filterWhen error delaying is not a
				// problem
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
					return r.getPredicate().apply(exchange);
				})
						// instead of immediately stopping main flux due to error, log and
						// swallow it
						.doOnError(e -> logger.error(
								"Error applying predicate for route: " + route.getId(),
								e))
						.onErrorResume(e -> Mono.empty()))
				// .defaultIfEmpty() put a static Route not found
				// or .switchIfEmpty()
				// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
				.next()
				// TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});

		/*
		 * TODO: trace logging if (logger.isTraceEnabled()) {
		 * logger.trace("RouteDefinition did not match: " + routeDefinition.getId()); }
		 */
	}
    
}

public abstract class AbstractHandlerMapping extends ApplicationObjectSupport
		implements HandlerMapping, Ordered, BeanNameAware {
    
	@Override
	public Mono<Object> getHandler(ServerWebExchange exchange) {
		return getHandlerInternal(exchange).map(handler -> {
			if (logger.isDebugEnabled()) {
				logger.debug(exchange.getLogPrefix() + "Mapped to " + handler);
			}
			if (CorsUtils.isCorsRequest(exchange.getRequest())) {
				CorsConfiguration configA = this.corsConfigurationSource.getCorsConfiguration(exchange);
				CorsConfiguration configB = getCorsConfiguration(handler, exchange);
				CorsConfiguration config = (configA != null ? configA.combine(configB) : configB);
				if (!getCorsProcessor().process(config, exchange) ||
						CorsUtils.isPreFlightRequest(exchange.getRequest())) {
					return REQUEST_HANDLED_HANDLER;
				}
			}
			return handler;
		});
	}	
}
```



## GatewayFilter

Spring Cloud Gateway的路径重写，限流，熔断等功能是如何实现的？这些功能又是如何接入到Spring WebFlux的呢？

Spring Cloud Gateway的重写，限流，熔断等功能是通过GatewayFilter来实现的，GatewayFilter又是通过GatewayFilterFactory来获取的

![image-20201117010726553](C:\study\study-notes\源码学习笔记\Spring Cloud Gateway源码解读\images\04.png)



通过GatewayFilterFactory来获取GatewayFilter，拿ModifyResponseBodyGatewayFilterFactory来讲：

```java
public interface GatewayFilterFactory<C> extends ShortcutConfigurable, Configurable<C> {
	...
	GatewayFilter apply(C config);
    ...
}
public class ModifyResponseBodyGatewayFilterFactory extends
		AbstractGatewayFilterFactory<ModifyResponseBodyGatewayFilterFactory.Config> {
	...
	@Override
	public GatewayFilter apply(Config config) {
		return new ModifyResponseGatewayFilter(config);
	}
    ...
}
```

这些GatewayFilterFactory的实现类都预先在GatewayAutoConfiguration中通过@Bean加入到Spring容器中了

那么GatewayFilter该如何接入Spring WebFlux呢？

通过RouteLocator逐步接入到Spring WebFlux中：

```java
public class GatewayAutoConfiguration {
	@Bean
	public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
			List<GatewayFilterFactory> GatewayFilters,
			List<RoutePredicateFactory> predicates,
			RouteDefinitionLocator routeDefinitionLocator,
			@Qualifier("webFluxConversionService") ConversionService conversionService) {
		return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates,
				GatewayFilters, properties, conversionService);
	}
}
```



## GlobalFilter

![image-20201118232408594](C:\study\study-notes\源码学习笔记\Spring Cloud Gateway源码解读\images\05)

GlobalFilter与GatewayFilterFactory类似，都是在GatewayAutoConfiguration中通过@Bean加入到Spring容器中

那么GlobalFilter该如何逐步接入到Spring WebFlux中呢？

通过FilteringWebHandler来接入：

```java
public class FilteringWebHandler implements WebHandler {
	private final List<GatewayFilter> globalFilters;

	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
		this.globalFilters = loadFilters(globalFilters);
	}

	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
		return filters.stream().map(filter -> {
			GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
			if (filter instanceof Ordered) {
				int order = ((Ordered) filter).getOrder();
				return new OrderedGatewayFilter(gatewayFilter, order);
			}
			return gatewayFilter;
		}).collect(Collectors.toList());
	}
}

public class GatewayAutoConfiguration {
	...
	@Bean
	public FilteringWebHandler filteringWebHandler(List<GlobalFilter> globalFilters) {
		return new FilteringWebHandler(globalFilters);
	}
    
	@Bean
	public RoutePredicateHandlerMapping routePredicateHandlerMapping(
			FilteringWebHandler webHandler, RouteLocator routeLocator,
			GlobalCorsProperties globalCorsProperties, Environment environment) {
        // 接入filteringWebHandler
		return new RoutePredicateHandlerMapping(webHandler, routeLocator,
				globalCorsProperties, environment);
	}      
    ...
}
```

至此GlobalFilter接入到FilteringWebHandler，然后FilteringWebHandler接入到HttpWebHandlerAdapter，HttpWebHandlerAdapter（WebHandler）接入到ReactorHttpHandlerAdapter，通过ReactorHttpHandlerAdapter接入到Reactor Netty



globalFilters又是如何应用的呢？

```java
public class FilteringWebHandler implements WebHandler {
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		// TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: " + combined);
		}

		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
    private static class DefaultGatewayFilterChain implements GatewayFilterChain {
		private final List<GatewayFilter> filters;
		...
		DefaultGatewayFilterChain(List<GatewayFilter> filters) {
			this.filters = filters;
			this.index = 0;
		}
        ...
        @Override
		public Mono<Void> filter(ServerWebExchange exchange) {
			return Mono.defer(() -> {
				if (this.index < filters.size()) {
					GatewayFilter filter = filters.get(this.index);
					DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this,
							this.index + 1);
					return filter.filter(exchange, chain);
				}
				else {
					return Mono.empty(); // complete
				}
			});
		}
    }
}
```



TODO：

- 实现调用的流程还未摸透



## Router的构造设计解读

Route在设计上，类似于Bean，通过BeanDefinition来生成，举个栗子，在Spring中，可以通过xml `<Bean>...<Bean/>`的形式来配置Bean，能生成对应的BeanDefinition，而Route是通过RouteDefinition来生成的，而RouterDefinition可以通过如下形式来进行配置：

```yaml
spring:
  application:
    name: gateway  
  cloud:
    gateway:
      routes:
      - id: service
        uri: lb://service
        predicates:
        - Path=/service/**
        filters:
        - StripPrefix=1
      - id: registry
        uri: lb://registry
        predicates:
        - Path=/registry/**
        filters:
        - StripPrefix=1
      - id: eureka
        uri: lb://registry
        predicates:
        - Path=/eureka/**
```

那么以上这种配置方式，又是如何加载至Spring Cloud Gateway中的呢？除此之外，还有哪些方式呢？

```java
public class GatewayAutoConfiguration {
	@Bean
	@ConditionalOnMissingBean
	public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(
			GatewayProperties properties) {
		return new PropertiesRouteDefinitionLocator(properties);
	}

	@Bean
	@ConditionalOnMissingBean(RouteDefinitionRepository.class)
	public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
		return new InMemoryRouteDefinitionRepository();
	}
}
```

PropertiesRouteDefinitionLocator、InMemoryRouteDefinitionRepository都是实现了RouteDefinitionLocator接口，所以会自动注入routeDefinitionLocator：

```java
public class GatewayAutoConfiguration {
	@Bean
	@Primary
	public RouteDefinitionLocator routeDefinitionLocator(
			List<RouteDefinitionLocator> routeDefinitionLocators) {
		return new CompositeRouteDefinitionLocator(
				Flux.fromIterable(routeDefinitionLocators));
	}
}
```

接下来的流程，就跟前面说到的一致了：

routeDefinitionLocator -》routeDefinitionRouteLocator -》 cachedCompositeRouteLocator -》routePredicateHandlerMapping -》DispatcherHandler -》HttpWebHandlerAdapter（WebHandler）-》ReactorHttpHandlerAdapter

另外一种配置router的方式：

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
	return builder.routes()
			.route("resource", r -> r.path("/resource")
				.filters(f -> f.filters(filterFactory.apply())
								.removeRequestHeader("Cookie")) // Prevents cookie being sent downstream
				.uri("http://resource:9000")) // Taking advantage of docker naming
			.build();
}
```

通过这种方式来配置RouteLocator的bean，会自动注入cachedCompositeRouteLocator，从而接入到Spring Cloud Gateway的生命周期中。 

Route是如何预先配置好的呢？

每个Route都会有其对应的的Builder，在这里是Route.AsyncBuilder，通过这个builder来贯穿整个Route的方法、参数设置

```java
public class RouteLocatorBuilder {
	public static class Builder {
		public Builder route(String id, Function<PredicateSpec, Route.AsyncBuilder> fn) {
			Route.AsyncBuilder routeBuilder = fn.apply(new RouteSpec(this).id(id));
			add(routeBuilder);
			return this;
		}    
    }
}
```

这里的`fn.apply(new RouteSpec(this).id(id))`对应的代码如下：

```java
r -> r.path("/resource")
	  .filters(f -> f.filters(filterFactory.apply())
               		 .removeRequestHeader("Cookie")) // Prevents cookie being sent downstream
	  .uri("http://resource:9000")
```

最后.uri返回的是AsyncBuilder

ps：详细过程，暂且不写，太过于复杂了，面对这种函数式接口编码风格，在看源码的过程中，需要以抽象的思维去看，去研究，不要一来就扎进细节，不然会困在代码的各种跳转中，无法自拔。在研究的过程中要辅于截图，这样能方便理解，记住出现函数式的地方，代码其实只是在配置，而非触发实现。



那么RouteDefinition是如何转换成route的呢？

```java
public class RouteDefinitionRouteLocator
		implements RouteLocator, BeanFactoryAware, ApplicationEventPublisherAware {
	...
	@Override
	public Flux<Route> getRoutes() {
		return this.routeDefinitionLocator.getRouteDefinitions().map(this::convertToRoute)
				// TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("RouteDefinition matched: " + route.getId());
					}
					return route;
				});

		/*
		 * TODO: trace logging if (logger.isTraceEnabled()) {
		 * logger.trace("RouteDefinition did not match: " + routeDefinition.getId()); }
		 */
	}
	// RouteDefinition转换为Route的核心方法：
	private Route convertToRoute(RouteDefinition routeDefinition) {
		AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
		List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

		return Route.async(routeDefinition).asyncPredicate(predicate)
				.replaceFilters(gatewayFilters).build();
	}

	private AsyncPredicate<ServerWebExchange> combinePredicates(
			RouteDefinition routeDefinition) {
		List<PredicateDefinition> predicates = routeDefinition.getPredicates();
		AsyncPredicate<ServerWebExchange> predicate = lookup(routeDefinition,
				predicates.get(0));

		for (PredicateDefinition andPredicate : predicates.subList(1,
				predicates.size())) {
			AsyncPredicate<ServerWebExchange> found = lookup(routeDefinition,
					andPredicate);
			predicate = predicate.and(found);
		}

		return predicate;
	}

	@SuppressWarnings("unchecked")
	private AsyncPredicate<ServerWebExchange> lookup(RouteDefinition route,
			PredicateDefinition predicate) {
		RoutePredicateFactory<Object> factory = this.predicates.get(predicate.getName());
		if (factory == null) {
			throw new IllegalArgumentException(
					"Unable to find RoutePredicateFactory with name "
							+ predicate.getName());
		}
		Map<String, String> args = predicate.getArgs();
		if (logger.isDebugEnabled()) {
			logger.debug("RouteDefinition " + route.getId() + " applying " + args + " to "
					+ predicate.getName());
		}

		Map<String, Object> properties = factory.shortcutType().normalize(args, factory,
				this.parser, this.beanFactory);
		Object config = factory.newConfig();
		ConfigurationUtils.bind(config, properties, factory.shortcutFieldPrefix(),
				predicate.getName(), validator, conversionService);
		if (this.publisher != null) {
			this.publisher.publishEvent(
					new PredicateArgsEvent(this, route.getId(), properties));
		}
		return factory.applyAsync(config);
	}    
}
```





