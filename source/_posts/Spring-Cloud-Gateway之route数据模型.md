title: Spring-Cloud-Gateway之route数据模型
author: Mr.G
tags:
  - 微服务
  - 网关
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 08:52:00
---
网关服务核心是将进入的请求正确合理的路由到下层具体的服务进行业务处理，由此可见网关服务的核心就是路由信息的构建。下面学习和阅读下Spring-Cloud-Gateway的route数据模型
<!-- more -->
---
首先查看Route模型的代码如下：

```
public class Route implements Ordered {

	/**
	 * 路由编号
	 * ID 编号，唯一
	 */
	private final String id;

	/**
	 * 路由向的 URI
	 *
	 */
	private final URI uri;

	/**
	 * 顺序
	 * 当请求匹配到多个路由时，使用顺序小的
	 */
	private final int order;

	/**
	 * 谓语数组
	 * 请求通过 predicates 判断是否匹配
	 */
	private final Predicate<ServerWebExchange> predicate;

	/**
	 * 过滤器数组
	 */
	private final List<GatewayFilter> gatewayFilters;
	}
	
```
> 由代码可以看到一个路由应该包含如下必要的信息：
> - id：路由编号，唯一
> - uri:路由向的 URI,对应的具体业务服务的URL
> - order:顺序，当请求匹配多个路由时，使用顺序小的
> - predicate: 请求匹配路由的断言条件
> - gatewayFilters: 当前路由上存在的过滤器，用于对请求做拦截处理

---
Route模型是通过RouteDefinition（路由定义）模型构建起来的，接下来查看RouteDefinition

```
/**
 * 路由定义实体信息，包含路由的定义信息
 * @author Spencer Gibb
 */
@Validated
public class RouteDefinition {

	/**
	 * 路由ID 编号，唯一
	 */
	@NotEmpty
	private String id = UUID.randomUUID().toString();

	/**
	 * 谓语定义数组
	 * predicates 属性，谓语定义数组
	 * 请求通过 predicates 判断是否匹配。在 Route 里，PredicateDefinition 转换成 Predicate
	 */
	@NotEmpty
	@Valid
	private List<PredicateDefinition> predicates = new ArrayList<>();

	/**
	 *过滤器定义数组
	 * filters 属性，过滤器定义数组。
	 * 在 Route 里，FilterDefinition 转换成 GatewayFilter
	 */
	@Valid
	private List<FilterDefinition> filters = new ArrayList<>();

	/**
	 * 路由指向的URI
	 */
	@NotNull
	private URI uri;

	/**
	 * 顺序
	 */
	private int order = 0;
}
```
这个模型与上面的Route模型是不是很相似，它是对route的定义以及描述，Spring-Cloud-Gateway最终会通过RouteDefinition来构建起Route实例信息。
细看RouteDefinition代码会发现其中包含两个数组分别是PredicateDefinition，FilterDefinition的数组。
> - PredicateDefinition : 断言条件(谓语)定义，构建 Route 时，PredicateDefinition 转换成 Predicate
> - FilterDefinition : 过滤条件的定义，构建Route 时，FilterDefinition 转换成 GatewayFilter

---
那么我们看下PredicateDefinition与FilterDefinition的代码

- PredicateDefinition
```
/**
 * 谓语定义,在 Route 里，PredicateDefinition将转换成 Predicate
 * @author Spencer Gibb
 */
@Validated
public class PredicateDefinition {
	/**
	 * 谓语定义名字
	 * 通过 name 对应到 org.springframework.cloud.gateway.handler.predicate.RoutePredicateFactory 的实现类。
	 * 例如: name=Query 对应到 QueryRoutePredicateFactory
	 */
	@NotNull
	private String name;
	/**
	 * 参数数组
	 * 例如，name=Host / args={"_genkey_0" : "iocoder.cn"} ，匹配请求的 hostname 为 iocoder.cn
	 */
	private Map<String, String> args = new LinkedHashMap<>();
}	
```
> PredicateDefinition 描述了构建Predicate的必要条件
> - name:名称，Spring-Cloud-Gateway会根据name找到Predicate的构建工厂类
> - args:参数，构建Predicate的参数

- FilterDefinition

```
/**
 * 过滤器定义，在 Route 里，FilterDefinition将转换成 GatewayFilter
 * @author Spencer Gibb
 */
@Validated
public class FilterDefinition {
	/**
	 * 过滤器定义名字
	 * 通过 name 对应到 org.springframework.cloud.gateway.filter.factory.GatewayFilterFactory 的实现类。
	 * 例如，name=AddRequestParameter 对应到 AddRequestParameterGatewayFilterFactory
	 */
	@NotNull
	private String name;


	/**
	 * 参数数组
	 * 例如 name=AddRequestParameter / args={"_genkey_0": "foo", "_genkey_1": "bar"} ，添加请求参数 foo 为 bar
	 */
	private Map<String, String> args = new LinkedHashMap<>();
}
```
> FilterDefinition 描述了构建GatewayFilter的必要条件
> - name:名称，Spring-Cloud-Gateway会根据name找到GatewayFilter的构建工厂类
> - args:参数，构建GatewayFilter的参数

---
通过这些基础的数据模型我可以清晰看到Spring-Cloud-Gateway构建路由的数据流向

```
graph LR
FilterDefinition-->RouteDefinition
PredicateDefinition-->RouteDefinition
RouteDefinition-->Route
```
通过数据流向可以帮助我们接下来阅读Route信息的初始化加载以及路由的使用

