---
title: Spring Security 
date: 2018-12-06 16:56:35
tags:
    - spring security
---

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
<!-- more -->
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

### Spring Security 认证流程

+ 用户使用用户名和密码登录  

+ 用户名密码被过滤器（默认为 UsernamePasswordAuthenticationFilter）获取到，配合其他权限信息（自定义），根据UsernamePasswordAuthenticationToken封装成一个未认证的Authentication（处在securityContext中）

+ Token（Authentication实现类）传递给 AuthenticationManager 进行认证

+ AuthenticationManager管理一系列的AuthenticationProvider，AuthenticationManager会遍历全部AuthenticationProvider去对Authentication进行认证

+ AuthenticationProvider会调用userDetailService去数据库中验证用户信息 ,返回一个userDetail对象（可自定义），认证成功后返回一个封装了用户权限信息的，以UsernamePasswordAuthenticationToken实现的带用户名和密码以及权限的Authentication 对象

+ 已经进行认证的Authentication返回到UsernamePasswordAuthenticationFilter中，如果验证失败，则进入unsuccessfulAuthentication;如果验证成功，则进入successfulAuthentication进行生成token