---
title: Spring Security 
date: 2018-12-06 16:56:35
tags:
    - spring security
---

### Spring Security 启动流程

+ @EnableWebSecurity注解配置启动spring secuiry,导入WebSecurityConfiguration配置
``` java 
	@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
	@Target(value = { java.lang.annotation.ElementType.TYPE })
	@Documented
	@Import({ WebSecurityConfiguration.class,
			SpringWebMvcImportSelector.class })
	@EnableGlobalAuthentication
	@Configuration
	public @interface EnableWebSecurity {

		/**
		* Controls debugging support for Spring Security. Default is false.
		* @return if true, enables debug support with Spring Security
		*/
		boolean debug() default false;
	}
```
<!-- more -->
+ WebSecurityConfiguration初始化WebSecurity，springSecurityFilterChain调用WebSecurity的build方法初始化各个Filter
``` java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}

```

+ webSecurity的build方法调用dobuild方法初始化，doBuild方法调用AbstractConfiguredSecurityBuilder的init方法
``` java 
	public final O build() throws Exception {
		if (this.building.compareAndSet(false, true)) {
			this.object = doBuild();
			return this.object;
		}
		throw new AlreadyBuiltException("This object has already been built");
	}

	@Override
	protected final O doBuild() throws Exception {
		synchronized (configurers) {
			buildState = BuildState.INITIALIZING;

			beforeInit();
			init();

			buildState = BuildState.CONFIGURING;

			beforeConfigure();
			configure();

			buildState = BuildState.BUILDING;

			O result = performBuild();

			buildState = BuildState.BUILT;

			return result;
		}
	}

```

+ AbstractConfiguredSecurityBuilder的init方法先去获取WebSecurityConfigurerAdapter，再调用WebSecurityConfigurerAdapter的init方法
``` java 
private void init() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

		for (SecurityConfigurer<O, B> configurer : configurers) {
			configurer.init((B) this);
		}

		for (SecurityConfigurer<O, B> configurer : configurersAddedInInitializing) {
			configurer.init((B) this);
		}
	}
```

+ WebSecurityConfigurerAdapter的init方法调用WebSecurityConfigurerAdapter的getHttp方法，创建HttpSecurity,HttpSecurity管理这各个XXXConfigurer,XXXConfigurer配置不同的过滤器
``` java 
public void init(final WebSecurity web) throws Exception {
		final HttpSecurity http = getHttp();
		web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
			public void run() {
				FilterSecurityInterceptor securityInterceptor = http
						.getSharedObject(FilterSecurityInterceptor.class);
				web.securityInterceptor(securityInterceptor);
			}
		});
}

protected final HttpSecurity getHttp() throws Exception {
		if (http != null) {
			return http;
		}

		DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
				.postProcess(new DefaultAuthenticationEventPublisher());
		localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

		AuthenticationManager authenticationManager = authenticationManager();
		authenticationBuilder.parentAuthenticationManager(authenticationManager);
		Map<Class<? extends Object>, Object> sharedObjects = createSharedObjects();

		http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
				sharedObjects);
		if (!disableDefaults) {
			// @formatter:off
			http
				.csrf().and()
				.addFilter(new WebAsyncManagerIntegrationFilter())
				.exceptionHandling().and()
				.headers().and()
				.sessionManagement().and()
				.securityContext().and()
				.requestCache().and()
				.anonymous().and()
				.servletApi().and()
				.apply(new DefaultLoginPageConfigurer<>()).and()
				.logout();
			// @formatter:on
			ClassLoader classLoader = this.context.getClassLoader();
			List<AbstractHttpConfigurer> defaultHttpConfigurers =
					SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

			for(AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
				http.apply(configurer);
			}
		}
		configure(http);
		return http;
}
```

+ getHttp方法除了自己会创建默认的一些XXXconfigurer,也会去创建我们自定义的configurer，最终将各个XXXconfigurer添加到configurersAddedInInitializing中
``` java 
private <C extends SecurityConfigurer<O, B>> void add(C configurer) throws Exception {
		Assert.notNull(configurer, "configurer cannot be null");

		Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer
				.getClass();
		synchronized (configurers) {
			if (buildState.isConfigured()) {
				throw new IllegalStateException("Cannot apply " + configurer
						+ " to already built object");
			}
			List<SecurityConfigurer<O, B>> configs = allowConfigurersOfSameType ? this.configurers
					.get(clazz) : null;
			if (configs == null) {
				configs = new ArrayList<SecurityConfigurer<O, B>>(1);
			}
			configs.add(configurer);
			this.configurers.put(clazz, configs);
			if (buildState.isInitializing()) {
				this.configurersAddedInInitializing.add(configurer);
			}
		}
	}
```
+ 最终会回到AbstractConfiguredSecurityBuilder的init方法，初始化XXXconfigurer,生成filter
``` java 
private void init() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

		for (SecurityConfigurer<O, B> configurer : configurers) {
			configurer.init((B) this);
		}

		for (SecurityConfigurer<O, B> configurer : configurersAddedInInitializing) {
			configurer.init((B) this);
		}
}
```

### Spring Security filter配置与原理

+ @EnableWebSecurity，加载 WebSecurityConfiguration 的配置
``` java 
    @Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
    @Target(value = { java.lang.annotation.ElementType.TYPE })
    @Documented
    @Import({ WebSecurityConfiguration.class,
            SpringWebMvcImportSelector.class })
    @EnableGlobalAuthentication
    @Configuration
    public @interface EnableWebSecurity {

        /**
        * Controls debugging support for Spring Security. Default is false.
        * @return if true, enables debug support with Spring Security
        */
        boolean debug() default false;
    }
```
+ 在WebSecurityConfiguration 中，会实例化 WebSecurity

``` java 
    @Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
			throws Exception {
		webSecurity = objectPostProcessor
				.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}

		Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);

		Integer previousOrder = null;
		Object previousConfig = null;
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
						"@Order on WebSecurityConfigurers must be unique. Order of "
								+ order + " was already used on " + previousConfig + ", so it cannot be used on "
								+ config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}
```
+ WebSecurity中可以通过addSecurityFilterChainBuilder() 方法，把各种filter的SecurityBuilder添加进来，而每个SecurityBuilder的泛型上界必须是SecurityFilterChain
``` java
    public WebSecurity addSecurityFilterChainBuilder(
			SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder) {
		this.securityFilterChainBuilders.add(securityFilterChainBuilder);
		return this;
	}

```
+ WebSecurity的performBuild() 方法实例化 FilterChainProxy，FilterChainProxy存储着SecurityFilterChain列表
``` java
    @Override
	protected Filter performBuild() throws Exception {
		Assert.state(
				!securityFilterChainBuilders.isEmpty(),
				"At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. More advanced users can invoke "
						+ WebSecurity.class.getSimpleName()
						+ ".addSecurityFilterChainBuilder directly");
		int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
		List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
				chainSize);
		for (RequestMatcher ignoredRequest : ignoredRequests) {
			securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
		}
		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
			securityFilterChains.add(securityFilterChainBuilder.build());
		}
		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
		if (httpFirewall != null) {
			filterChainProxy.setFirewall(httpFirewall);
		}
		filterChainProxy.afterPropertiesSet();

		Filter result = filterChainProxy;
		if (debugEnabled) {
			logger.warn("\n\n"
					+ "********************************************************************\n"
					+ "**********        Security debugging is enabled.       *************\n"
					+ "**********    This may include sensitive information.  *************\n"
					+ "**********      Do not use in a production system!     *************\n"
					+ "********************************************************************\n\n");
			result = new DebugFilter(filterChainProxy);
		}
		postBuildAction.run();
		return result;
	}
    
```
+ WebSecurity的performBuild()方法中将securityFilterChainBuilders列表里面的每个SecurityBuilder对象构建SecurityFilterChain实例，将SecurityFilterChain列表作为参数构建FilterChainProxy,SecurityFilterChain接口定义了两个方法，一个是存储filter，一个是uri规则匹配
``` java
    public interface SecurityFilterChain {

	boolean matches(HttpServletRequest request);

	List<Filter> getFilters();
}
```

+ 请求到达的时候，FilterChainProxy的dofilter()方法，会遍历所有的SecurityFilterChain，对匹配到的url，则一一调用SecurityFilterChain中的filter做认证授权。FilterChainProxy的dofilter()中调用了doFilterInternal()方法，如下：
``` java
    private void doFilterInternal(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {

		FirewalledRequest fwRequest = firewall
				.getFirewalledRequest((HttpServletRequest) request);
		HttpServletResponse fwResponse = firewall
				.getFirewalledResponse((HttpServletResponse) response);

		List<Filter> filters = getFilters(fwRequest);

		if (filters == null || filters.size() == 0) {
			if (logger.isDebugEnabled()) {
				logger.debug(UrlUtils.buildRequestUrl(fwRequest)
						+ (filters == null ? " has no matching filters"
								: " has an empty filter list"));
			}

			fwRequest.reset();

			chain.doFilter(fwRequest, fwResponse);

			return;
		}

		VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
		vfc.doFilter(fwRequest, fwResponse);
	}

	/**
	 * Returns the first filter chain matching the supplied URL.
	 *
	 * @param request the request to match
	 * @return an ordered array of Filters defining the filter chain
	 */
	private List<Filter> getFilters(HttpServletRequest request) {
		for (SecurityFilterChain chain : filterChains) {
			if (chain.matches(request)) {
				return chain.getFilters();
			}
		}

		return null;
	}
```