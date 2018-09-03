title: Spring-Cloud-Gateway之请求处理流程
author: Mr.G
tags:
  - 微服务
  - 网关
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 01:04:00
---
Spring-Cloud-Gateway 初始化，路由模型，以及路由加载等源码在上几篇学习文档中已经描述，接下来来看Spring-Cloud-Gateway是怎么通过这些来对我们的请求进行路由处理的

---
<!-- more -->
> Spring-Cloud-Gateway整体流程图

![image](https://raw.githubusercontent.com/guofazhan/image/master/gateway.png)

> - DispatcherHandler：所有请求的调度器，负载请求分发
> - RoutePredicateHandlerMapping:路由谓语匹配器，用于路由的查找，以及找到路由后返回对应的WebHandler，DispatcherHandler会依次遍历HandlerMapping集合进行处理
> - FilteringWebHandler : 使用Filter链表处理请求的WebHandler，RoutePredicateHandlerMapping找到路由后返回对应的FilteringWebHandler对请求进行处理，FilteringWebHandler负责组装Filter链表并调用链表处理请求。

---
1. DispatcherHandler

```java
org.springframework.web.reactive.DispatcherHandler

	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		if (logger.isDebugEnabled()) {
			ServerHttpRequest request = exchange.getRequest();
			logger.debug("Processing " + request.getMethodValue() + " request for [" + request.getURI() + "]");
		}
		//校验handlerMapping集合是否为空
		if (this.handlerMappings == null) {
			return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
		}
		//依次遍历handlerMapping集合进行请求处理
		return Flux.fromIterable(this.handlerMappings)
				.concatMap(mapping ->
				//通过mapping获取mapping对应的handler
				mapping.getHandler(exchange))
				.next()
				.switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
				.flatMap(handler -> 
				//调用handler处理
				invokeHandler(exchange, handler))
				.flatMap(result -> handleResult(exchange, result));
	}
	
		private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
			//判断当前handlerAdapter与handler是否匹配
				if (handlerAdapter.supports(handler)) {
					return handlerAdapter.handle(exchange, handler);
				}
			}
		}
		return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
	}

```
> DispatcherHandler的handler执行顺序
> - 校验handlerMapping
> - 遍历Mapping获取mapping对应的handler(此处会找到gateway对应的 RoutePredicateHandlerMapping，并通过 RoutePredicateHandlerMapping获取handler（FilteringWebHandler）)
> - 通过handler对应的HandlerAdapter对handler进行调用（gateway使用的 SimpleHandlerAdapter） 即 FilteringWebHandler与SimpleHandlerAdapter对应

```java
org.springframework.web.reactive.result.SimpleHandlerAdapter
	public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
		WebHandler webHandler = (WebHandler) handler;
		//调用handler的handle方法处理请求
		Mono<Void> mono = webHandler.handle(exchange);
		return mono.then(Mono.empty());
	}
```

2. RoutePredicateHandlerMapping

```java
public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

	private final FilteringWebHandler webHandler;
	private final RouteLocator routeLocator;

	public RoutePredicateHandlerMapping(FilteringWebHandler webHandler, RouteLocator routeLocator) {
		this.webHandler = webHandler;
		this.routeLocator = routeLocator;

        //设置排序字段1，此处的目的是Spring Cloud Gateway 的 GatewayWebfluxEndpoint 提供 HTTP API ，不需要经过网关
        //它通过 RequestMappingHandlerMapping 进行请求匹配处理。RequestMappingHandlerMapping 的 order = 0 ，需要排在 RoutePredicateHandlerMapping 前面。所有，RoutePredicateHandlerMapping 设置 order = 1 。
		setOrder(1);
	}

	@Override
	protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		//设置mapping到上下文环境
		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getClass().getSimpleName());

		//查找路由
		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}

					//将找到的路由信息设置到上下文环境中
					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
					//返回mapping对应的WebHandler即FilteringWebHandler
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					//当前未找到路由时返回空，并移除GATEWAY_PREDICATE_ROUTE_ATTR
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
					}
				})));
	}

	protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		//通过路由定位器获取路由信息
		return this.routeLocator.getRoutes()
				.filter(route -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, route.getId());
					//返回通过谓语过滤的路由信息
					return route.getPredicate().test(exchange);
				})
				// .defaultIfEmpty() put a static Route not found
				// or .switchIfEmpty()
				// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
				.next()
				//TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					//校验路由，目前空实现
					validateRoute(route, exchange);
					return route;
				});

	}
}
```
> RoutePredicateHandlerMapping的执行顺序
> - 通过路由定位器获取全部路由（RouteLocator）
> - 通过路由的谓语（Predicate）过滤掉不可用的路由信息
> - 查找到路由信息后将路由信息设置当上下文环境中（GATEWAY_ROUTE_ATTR）
> - 返回gatway自定的webhandler（FilteringWebHandler）

> 备注：
> 在构建方法中看到setOrder(1);作用：Spring Cloud Gateway 的 GatewayWebfluxEndpoint 提供 HTTP API ，不需要经过网关。
        通过 RequestMappingHandlerMapping 进行请求匹配处理。RequestMappingHandlerMapping 的 order = 0 ，需要排在 RoutePredicateHandlerMapping 前面，所以设置 order = 1 。


3. FilteringWebHandler

```java
/**
 * 通过过滤器处理web请求的处理器
 * WebHandler that delegates to a chain of {@link GlobalFilter} instances and
 * {@link GatewayFilterFactory} instances then to the target {@link WebHandler}.
 *
 * @author Rossen Stoyanchev
 * @author Spencer Gibb
 * @since 0.1
 */
public class FilteringWebHandler implements WebHandler {
	protected static final Log logger = LogFactory.getLog(FilteringWebHandler.class);

	/**
	 * 全局过滤器
	 */
	private final List<GatewayFilter> globalFilters;

	public FilteringWebHandler(List<GlobalFilter> globalFilters) {
		this.globalFilters = loadFilters(globalFilters);
	}

	/**
	 * 包装加载全局的过滤器，将全局过滤器包装成GatewayFilter
	 * @param filters
	 * @return
	 */
	private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
		return filters.stream()
				.map(filter -> {
					//将所有的全局过滤器包装成网关过滤器
					GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
					//判断全局过滤器是否实现了可排序接口
					if (filter instanceof Ordered) {
						int order = ((Ordered) filter).getOrder();
						//包装成可排序的网关过滤器
						return new OrderedGatewayFilter(gatewayFilter, order);
					}
					return gatewayFilter;
				}).collect(Collectors.toList());
	}


	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		//获取请求上下文设置的路由实例
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		//获取路由定义下的网关过滤器集合
		List<GatewayFilter> gatewayFilters = route.getFilters();

		//组合全局的过滤器与路由配置的过滤器
		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		//添加路由配置过滤器到集合尾部
		combined.addAll(gatewayFilters);
		//对过滤器进行排序
		//TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		logger.debug("Sorted gatewayFilterFactories: "+ combined);
		//创建过滤器链表对其进行链式调用
		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
}
```
> FilteringWebHandler的执行顺序
> - 构建一个包含全局过滤器的集合（combined）
> - 获取上下中的路由信息GATEWAY_ROUTE_ATTR
> - 将路由里的过滤器添加到集合中（combined）
> - 对过滤器集合进行排序操作
> - 通过过滤器集合组装过滤器链表，并进行调用（DefaultGatewayFilterChain与Servlet中的FilterChain与原理是一致的）
> - 通过过滤器来处理请求到具体业务服务


---
整个代码阅读下来，使得对Spring-Cloud-Gateway的工作原理清晰明了的理解。



