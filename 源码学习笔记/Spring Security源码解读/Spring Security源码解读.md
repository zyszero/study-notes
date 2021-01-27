# Spring Security 源码解读


## Spring Security 4.x源码解读
如何从零构造一个Spring Security框架？（或者构造一个权限框架）

核心是过滤器，设置一个请求处理入口，即“门卫”，不同用户需要应用不同的过滤规则，所以要有个过滤规则管理器，即“门禁系统”，“门卫”管控“门禁系统”   用户对应状态的管理。



### 鉴权全流程

核心接口org.springframework.security.config.annotation.SecurityConfigurer的实现

```java
public final class HttpBasicConfigurer<B extends HttpSecurityBuilder<B>> extends
		AbstractHttpConfigurer<HttpBasicConfigurer<B>, B> {
	@Override
	public void configure(B http) throws Exception {
        // 可看做为门禁系统，即一些权限鉴定的管理
		AuthenticationManager authenticationManager = http
				.getSharedObject(AuthenticationManager.class);
        // 基础过滤器，看做为门卫
		BasicAuthenticationFilter basicAuthenticationFilter = new BasicAuthenticationFilter(
				authenticationManager, this.authenticationEntryPoint);
		if (this.authenticationDetailsSource != null) {
			basicAuthenticationFilter
					.setAuthenticationDetailsSource(this.authenticationDetailsSource);
		}
		RememberMeServices rememberMeServices = http.getSharedObject(RememberMeServices.class);
		if(rememberMeServices != null) {
			basicAuthenticationFilter.setRememberMeServices(rememberMeServices);
		}
		basicAuthenticationFilter = postProcess(basicAuthenticationFilter);
		http.addFilter(basicAuthenticationFilter);
	}    
}
```

org.springframework.security.authentication.AuthenticationManager 如何生成？

通过核心方法： AuthenticationManagerBuilder#performBuild

```java
	@Override
	protected ProviderManager performBuild() throws Exception {
		if (!isConfigured()) {
			logger.debug("No authenticationProviders and no parentAuthenticationManager defined. Returning null.");
			return null;
		}
        // authenticationProviders为权限规则
		ProviderManager providerManager = new ProviderManager(authenticationProviders,
				parentAuthenticationManager);
		if (eraseCredentials != null) {
			providerManager.setEraseCredentialsAfterAuthentication(eraseCredentials);
		}
		if (eventPublisher != null) {
			providerManager.setAuthenticationEventPublisher(eventPublisher);
		}
		providerManager = postProcess(providerManager);
		return providerManager;
	}
```

“门卫”如何接入servlet容器中？

接入Servlet容器，即要实现Filter接口，BasicAuthenticationFilter实现了Filter接口，这里我们关注核心方法org.springframework.web.filter.OncePerRequestFilter#doFilter

```java
/**
 * This {@code doFilter} implementation stores a request attribute for
 * "already filtered", proceeding without filtering again if the
 * attribute is already there.
 * @see #getAlreadyFilteredAttributeName
 * @see #shouldNotFilter
 * @see #doFilterInternal
 */
@Override
public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
		throws ServletException, IOException {
	if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
		throw new ServletException("OncePerRequestFilter just supports HTTP requests");
	}
    // 将ServletRequest、ServletResponse 转换为HttpServletRequest和HttpServletResponse
	HttpServletRequest httpRequest = (HttpServletRequest) request;
	HttpServletResponse httpResponse = (HttpServletResponse) response;
	...
	if (hasAlreadyFilteredAttribute || skipDispatch(httpRequest) || shouldNotFilter(httpRequest)) {
		// Proceed without invoking this filter...
		filterChain.doFilter(request, response);
	}
	else {
		// Do invoke this filter...
		request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
		try {
			doFilterInternal(httpRequest, httpResponse, filterChain);
		}
		finally {
			// Remove the "already filtered" request attribute for this request.
			request.removeAttribute(alreadyFilteredAttributeName);
		}
	}
}

@Override
protected void doFilterInternal(HttpServletRequest request,
		HttpServletResponse response, FilterChain chain)
				throws IOException, ServletException {
	final boolean debug = this.logger.isDebugEnabled();
    // 校验HttpHeader,须符合指定规则
	String header = request.getHeader("Authorization");
	if (header == null || !header.startsWith("Basic ")) {
		chain.doFilter(request, response);
		return;
	}
	try {
        // 提取token
		String[] tokens = extractAndDecodeHeader(header, request);
		...
		if (authenticationIsRequired(username)) {
			UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
					username, tokens[1]);
			authRequest.setDetails(
					this.authenticationDetailsSource.buildDetails(request));
            // 交由鉴定管理器进行鉴定，核心
			Authentication authResult = this.authenticationManager
					.authenticate(authRequest);
			if (debug) {
				this.logger.debug("Authentication success: " + authResult);
			}
            // SecurityContextHolder的设计也可以借鉴学习一下，本质上就是利用ThreadLocal来实现的
			SecurityContextHolder.getContext().setAuthentication(authResult);
			this.rememberMeServices.loginSuccess(request, response, authResult);
			onSuccessfulAuthentication(request, response, authResult);
		}
	}
	catch (AuthenticationException failed) {
		SecurityContextHolder.clearContext();
		if (debug) {
			this.logger.debug("Authentication request for failed: " + failed);
		}
		this.rememberMeServices.loginFail(request, response);
		onUnsuccessfulAuthentication(request, response, failed);
		if (this.ignoreFailure) {
			chain.doFilter(request, response);
		}
		else {
			this.authenticationEntryPoint.commence(request, response, failed);
		}
		return;
	}
	chain.doFilter(request, response);
}
```

设计学习:

org.springframework.security.authentication.AuthenticationProvider： 

```java
public interface AuthenticationProvider {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);    
}
```



org.springframework.security.core.context.SecurityContextHolder：

ThreadLocal的策略写法？

```java
public class SecurityContextHolder {
	static {
		initialize();
	}

	private static void initialize() {
		if (strategyName.equals(MODE_THREADLOCAL)) {
			strategy = new ThreadLocalSecurityContextHolderStrategy();
		}
		else if (strategyName.equals(MODE_INHERITABLETHREADLOCAL)) {
			strategy = new InheritableThreadLocalSecurityContextHolderStrategy();
		}
		else if (strategyName.equals(MODE_GLOBAL)) {
			strategy = new GlobalSecurityContextHolderStrategy();
		}
		else {
			// Try to load a custom strategy
			try {
				Class<?> clazz = Class.forName(strategyName);
				Constructor<?> customStrategy = clazz.getConstructor();
				strategy = (SecurityContextHolderStrategy) customStrategy.newInstance();
			}
			catch (Exception ex) {
				ReflectionUtils.handleReflectionException(ex);
			}
		}

		initializeCount++;
	}
}

public interface SecurityContextHolderStrategy {
	// ~ Methods


	/**
	 * Clears the current context.
	 */
	void clearContext();

	/**
	 * Obtains the current context.
	 *
	 * @return a context (never <code>null</code> - create a default implementation if
	 * necessary)
	 */
	SecurityContext getContext();

	/**
	 * Sets the current context.
	 *
	 * @param context to the new argument (should never be <code>null</code>, although
	 * implementations must check if <code>null</code> has been passed and throw an
	 * <code>IllegalArgumentException</code> in such cases)
	 */
	void setContext(SecurityContext context);

	/**
	 * Creates a new, empty context implementation, for use by
	 * <tt>SecurityContextRepository</tt> implementations, when creating a new context for
	 * the first time.
	 *
	 * @return the empty context.
	 */
	SecurityContext createEmptyContext();
}

final class ThreadLocalSecurityContextHolderStrategy implements
		SecurityContextHolderStrategy {
	// ~ Static fields/initializers


	private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<SecurityContext>();
    ...
}

final class GlobalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {
	// ~ Static fields/initializers
	

	private static SecurityContext contextHolder;
    ...
}

final class InheritableThreadLocalSecurityContextHolderStrategy implements
		SecurityContextHolderStrategy {
	// ~ Static fields/initializers
	

	private static final ThreadLocal<SecurityContext> contextHolder = new InheritableThreadLocal<SecurityContext>();
    ...
}
```



如何鉴权？

拿org.springframework.security.authentication.ProviderManager来举例：

其中

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware,
		InitializingBean {

	public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		Authentication result = null;
		boolean debug = logger.isDebugEnabled();
        // AuthenticationProvider接口的设计，值得借鉴
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}
			...
			try {
                // 核心，拿AbstractUserDetailsAuthenticationProvider举例
				result = provider.authenticate(authentication);

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException e) {
				prepareException(e, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw e;
			}
			catch (InternalAuthenticationServiceException e) {
				prepareException(e, authentication);
				throw e;
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}
		....
	}	
}
```



org.springframework.security.authentication.dao.AbstractUserDetailsAuthenticationProvider：

~~~java
public Authentication authenticate(Authentication authentication)
		throws AuthenticationException {
	Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
			messages.getMessage(
					"AbstractUserDetailsAuthenticationProvider.onlySupports",
					"Only UsernamePasswordAuthenticationToken is supported"));
	// Determine username
	String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
			: authentication.getName();
	boolean cacheWasUsed = true;
	UserDetails user = this.userCache.getUserFromCache(username);
	if (user == null) {
		cacheWasUsed = false;
		try {
			user = retrieveUser(username,
					(UsernamePasswordAuthenticationToken) authentication);
		}
		catch (UsernameNotFoundException notFound) {
			logger.debug("User '" + username + "' not found");
			if (hideUserNotFoundExceptions) {
				throw new BadCredentialsException(messages.getMessage(
						"AbstractUserDetailsAuthenticationProvider.badCredentials",
						"Bad credentials"));
			}
			else {
				throw notFound;
			}
		}
		Assert.notNull(user,
				"retrieveUser returned null - a violation of the interface contract");
	}
	try {
		preAuthenticationChecks.check(user);
		additionalAuthenticationChecks(user,
				(UsernamePasswordAuthenticationToken) authentication);
	}
	catch (AuthenticationException exception) {
		if (cacheWasUsed) {
			// There was a problem, so try again after checking
			// we're using latest data (i.e. not from the cache)
			cacheWasUsed = false;
			user = retrieveUser(username,
					(UsernamePasswordAuthenticationToken) authentication);
			preAuthenticationChecks.check(user);
			additionalAuthenticationChecks(user,
					(UsernamePasswordAuthenticationToken) authentication);
		}
		else {
			throw exception;
		}
	}
	postAuthenticationChecks.check(user);
	if (!cacheWasUsed) {
		this.userCache.putUserInCache(user);
	}
	Object principalToReturn = user;
	if (forcePrincipalAsString) {
		principalToReturn = user.getUsername();
	}
    // 创建token
	return createSuccessAuthentication(principalToReturn, authentication, user);
}
~~~



ProviderManager从哪来？

```java
public class AuthenticationManagerFactoryBean implements
		FactoryBean<AuthenticationManager>, BeanFactoryAware {
	public AuthenticationManager getObject() throws Exception {
		try {
			return (AuthenticationManager) bf.getBean(BeanIds.AUTHENTICATION_MANAGER);
		}
		catch (NoSuchBeanDefinitionException e) {
			if (BeanIds.AUTHENTICATION_MANAGER.equals(e.getBeanName())) {
				try {
					UserDetailsService uds = bf.getBean(UserDetailsService.class);
					DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
					provider.setUserDetailsService(uds);
					provider.afterPropertiesSet();
					return new ProviderManager(
							Arrays.<AuthenticationProvider> asList(provider));
				}
				catch (NoSuchBeanDefinitionException noUds) {
				}
				throw new NoSuchBeanDefinitionException(BeanIds.AUTHENTICATION_MANAGER,
						MISSING_BEAN_ERROR_MESSAGE);
			}
			throw e;
		}
	}	
}
```

AuthenticationManagerFactoryBean从哪来？

```java
public class HttpSecurityBeanDefinitionParser implements BeanDefinitionParser {
	private BeanReference createAuthenticationManager(Element element, ParserContext pc,
			ManagedList<BeanReference> authenticationProviders) {
		..

		if (StringUtils.hasText(parentMgrRef)) {
			...
		}
		else {
            // 核心，AuthenticationManagerFactoryBean从这里产生
			RootBeanDefinition amfb = new RootBeanDefinition(
					AuthenticationManagerFactoryBean.class);
			amfb.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			String amfbId = pc.getReaderContext().generateBeanName(amfb);
			pc.registerBeanComponent(new BeanComponentDefinition(amfb, amfbId));
			RootBeanDefinition clearCredentials = new RootBeanDefinition(
					MethodInvokingFactoryBean.class);
			clearCredentials.getPropertyValues().addPropertyValue("targetObject",
					new RuntimeBeanReference(amfbId));
			clearCredentials.getPropertyValues().addPropertyValue("targetMethod",
					"isEraseCredentialsAfterAuthentication");

			authManager.addConstructorArgValue(new RuntimeBeanReference(amfbId));
			authManager.addPropertyValue("eraseCredentialsAfterAuthentication",
					clearCredentials);
		}

		authManager.getRawBeanDefinition().setSource(pc.extractSource(element));
		BeanDefinition authMgrBean = authManager.getBeanDefinition();
		String id = pc.getReaderContext().generateBeanName(authMgrBean);
		pc.registerBeanComponent(new BeanComponentDefinition(authMgrBean, id));

		return new RuntimeBeanReference(id);    	
    }
}
```



org.springframework.web.filter.GenericFilterBean

```java
public abstract class GenericFilterBean implements
		Filter, BeanNameAware, EnvironmentAware, ServletContextAware, InitializingBean, DisposableBean {
	....
}
```

通过Filter接口接入Servlet，通过InitializingBean, DisposableBean接入Spring容器的生命周期



### Authentication

Authentication认证信息的承载者，设计如下：

```java
public interface Authentication extends Principal, Serializable {
	...
}
```

拿UsernamePasswordAuthenticationToken来举例：

```java
public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {
	private final Object principal;
	private Object credentials;
	...	
}
```



## Spring Security 5.3 源码解读

Spring Security 如何接入Spring WebFlux？

接入逻辑与Spring Cloud Gateway 一样，可以参考《Spring Cloud Gateway的源码解读》，本质上是Filter接入Spring WebFlux，即通过WebFliter接口入手。“门卫” =》WebFilterChainProxy

那么4.x中的AuthenticationManager对应5.3中，又是以什么形式出现呢？

拿org.springframework.security.config.web.server.ServerHttpSecurity.HttpBasicSpec来看：

```java
public class ServerHttpSecurity {

	private List<WebFilter> webFilters = new ArrayList<>();    
    ...
	public ServerHttpSecurity addFilterAt(WebFilter webFilter, SecurityWebFiltersOrder order) {
		this.webFilters.add(new OrderedWebFilter(webFilter, order.getOrder()));
		return this;
	}        
    ...
	public class HttpBasicSpec {
		private ReactiveAuthenticationManager authenticationManager;
        ...
		protected void configure(ServerHttpSecurity http) {
			MediaTypeServerWebExchangeMatcher restMatcher = new MediaTypeServerWebExchangeMatcher(
				MediaType.APPLICATION_ATOM_XML,
				MediaType.APPLICATION_FORM_URLENCODED, MediaType.APPLICATION_JSON,
				MediaType.APPLICATION_OCTET_STREAM, MediaType.APPLICATION_XML,
				MediaType.MULTIPART_FORM_DATA, MediaType.TEXT_XML);
			restMatcher.setIgnoredMediaTypes(Collections.singleton(MediaType.ALL));
			ServerHttpSecurity.this.defaultEntryPoints.add(new DelegateEntry(restMatcher, this.entryPoint));
			// 核心是AuthenticationWebFilter
            AuthenticationWebFilter authenticationFilter = new AuthenticationWebFilter(
				this.authenticationManager);
			authenticationFilter.setAuthenticationFailureHandler(new ServerAuthenticationEntryPointFailureHandler(this.entryPoint));
			authenticationFilter.setAuthenticationConverter(new ServerHttpBasicAuthenticationConverter());
			authenticationFilter.setSecurityContextRepository(this.securityContextRepository);
			http.addFilterAt(authenticationFilter, SecurityWebFiltersOrder.HTTP_BASIC);
		}            
	}
    
	public SecurityWebFilterChain build() {
    	...
		AnnotationAwareOrderComparator.sort(this.webFilters);
		List<WebFilter> sortedWebFilters = new ArrayList<>();
		this.webFilters.forEach( f -> {
			if (f instanceof OrderedWebFilter) {
				f = ((OrderedWebFilter) f).webFilter;
			}
			sortedWebFilters.add(f);
		});
		sortedWebFilters.add(0, new ServerWebExchangeReactorContextWebFilter());
		return new MatcherSecurityWebFilterChain(getSecurityMatcher(), sortedWebFilters);            
    }
}
```



```java
public class MatcherSecurityWebFilterChain implements SecurityWebFilterChain {
	...
}
```

就如例子中的：

```java
  @Bean
  public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
    http.csrf()
        .disable()
        .authorizeExchange()
        .matchers(PathRequest.toStaticResources().atCommonLocations())
        .permitAll()
        .matchers(EndpointRequest.to("health"))
        .permitAll()
        .matchers(EndpointRequest.to("info"))
        .permitAll()
        .matchers(EndpointRequest.toAnyEndpoint())
        .hasRole(Role.LIBRARY_ADMIN.name())
        .pathMatchers(HttpMethod.POST, "/books/{bookId}/borrow")
        .hasRole(Role.LIBRARY_USER.name())
        .pathMatchers(HttpMethod.POST, "/books/{bookId}/return")
        .hasRole(Role.LIBRARY_USER.name())
        .pathMatchers(HttpMethod.POST, "/books")
        .hasRole(Role.LIBRARY_CURATOR.name())
        .pathMatchers(HttpMethod.DELETE, "/books")
        .hasRole(Role.LIBRARY_CURATOR.name())
        .pathMatchers("/users/**")
        .hasRole(Role.LIBRARY_ADMIN.name())
        .anyExchange()
        .authenticated()
        .and()
        .oauth2ResourceServer()
        .jwt()
        .jwtAuthenticationConverter(libraryUserJwtAuthenticationConverter());
    // .jwtAuthenticationConverter(libraryUserRolesJwtAuthenticationConverter());
    return http.build();
  }
```



org.springframework.security.web.server.authentication.AuthenticationWebFilter：

```java
public class AuthenticationWebFilter implements WebFilter {
	@Override
	public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
		return this.requiresAuthenticationMatcher.matches(exchange)
			.filter( matchResult -> matchResult.isMatch())
             // this.authenticationConverter.convert(exchange) 相当于4.x的提取token
			.flatMap( matchResult -> this.authenticationConverter.convert(exchange))
			.switchIfEmpty(chain.filter(exchange).then(Mono.empty()))
             // 这里进行鉴权 
			.flatMap( token -> authenticate(exchange, chain, token))
			.onErrorResume(AuthenticationException.class, e -> this.authenticationFailureHandler
					.onAuthenticationFailure(new WebFilterExchange(exchange, chain), e));
	}

	private Mono<Void> authenticate(ServerWebExchange exchange, WebFilterChain chain, Authentication token) {
		return this.authenticationManagerResolver.resolve(exchange)
			.flatMap(authenticationManager -> authenticationManager.authenticate(token))
			.switchIfEmpty(Mono.defer(() -> Mono.error(new IllegalStateException("No provider found for " + token.getClass()))))
			.flatMap(authentication -> onAuthenticationSuccess(authentication, new WebFilterExchange(exchange, chain)));
	}    
}

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


通过搭载FilteringWebHandler，来实现接入Spring Security



org.springframework.security.web.server.WebFilterChainProxy

```java
public class WebFilterChainProxy implements WebFilter {
    // 可以看做为每个公司特有的FilterChain,这个FilterChain中包含着特有的Filter链
	private final List<SecurityWebFilterChain> filters;

	public WebFilterChainProxy(List<SecurityWebFilterChain> filters) {
		this.filters = filters;
	}

	public WebFilterChainProxy(SecurityWebFilterChain... filters) {
		this.filters = Arrays.asList(filters);
	}

	@Override
	public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
		return Flux.fromIterable(this.filters)
				.filterWhen( securityWebFilterChain -> securityWebFilterChain.matches(exchange))
				.next()
				.switchIfEmpty(chain.filter(exchange).then(Mono.empty()))
				.flatMap( securityWebFilterChain -> securityWebFilterChain.getWebFilters()
					.collectList()
				)
            	// 注意：这里的webHandler指的是ServerWebExchange
				.map( filters -> new FilteringWebHandler(webHandler -> chain.filter(webHandler), filters))
				.map( handler -> new DefaultWebFilterChain(handler) )
				.flatMap( securedChain -> securedChain.filter(exchange));
	}
}
```



搞清楚 FilteringWebHandler =》 DefaultWebFilterChain 

以及DefaultWebFilterChain如何自身形成链

```java
public class DefaultWebFilterChain implements WebFilterChain {
	public DefaultWebFilterChain(WebHandler handler, List<WebFilter> filters) {
		Assert.notNull(handler, "WebHandler is required");
		this.allFilters = Collections.unmodifiableList(filters);
		this.handler = handler;
		DefaultWebFilterChain chain = initChain(filters, handler);
		this.currentFilter = chain.currentFilter;
		this.chain = chain.chain;
	}

    // 形成DefaultWebFilterChain链条，通过不断组合，来实现链，即每个WebFilter对应生成一个DefaultWebFilterChain，而每个DefaultWebFilterChain持有上一个DefaultWebFilterChain，从而实现filter链
	private static DefaultWebFilterChain initChain(List<WebFilter> filters, WebHandler handler) {
		DefaultWebFilterChain chain = new DefaultWebFilterChain(filters, handler, null, null);
		ListIterator<? extends WebFilter> iterator = filters.listIterator(filters.size());
		while (iterator.hasPrevious()) {
			chain = new DefaultWebFilterChain(filters, handler, iterator.previous(), chain);
		}
		return chain;
	}
    
    ...
	@Override
	public Mono<Void> filter(ServerWebExchange exchange) {
		return Mono.defer(() ->
				this.currentFilter != null && this.chain != null ?
						invokeFilter(this.currentFilter, this.chain, exchange) :
						this.handler.handle(exchange));
	}

	private Mono<Void> invokeFilter(WebFilter current, DefaultWebFilterChain chain, ServerWebExchange exchange) {
		String currentName = current.getClass().getName();
		return current.filter(exchange, chain).checkpoint(currentName + " [DefaultWebFilterChain]");
	}    
}
```



TODO：要再整理整理，对比下4.x版本的源码流程，应从@Bean配置这个点切入，然后接上Spring Security的生命周期，从而了解清楚Spring Security如何接入Spring WebFlux，如何接入Spring容器



org.springframework.security.web.access.intercept.FilterSecurityInterceptor