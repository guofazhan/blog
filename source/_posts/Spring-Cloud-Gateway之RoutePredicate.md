title: Spring-Cloud-Gateway之RoutePredicate
author: Mr.G
tags:
  - 网关
  - 微服务
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 16:57:00
---
Spring-Cloud-Gateway路由选择是通过Predicate函数式接口进行判断当前路由是否满足给定条件。下来阅读Spring-Cloud-Gateway路由的Predicate构建流程以及选择路由的源码

---
<!-- more -->
首先看下JDK8中Predicate接口定义
> Predicate<T> 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将Predicate组合成其他复杂的逻辑（比如：与，或，非）。可以用于接口请求参数校验、判断新老数据是否有变化需要进行更新操作。add--与、or--或、negate--非
> -  boolean test(T t); 判断
> -  Predicate<T> and(Predicate<? super T> other) 接收一个Predicate类型，也就是将传入的条件和当前条件以并且(AND)的关系组合
> - Predicate<T> or(Predicate<? super T> other)接收一个Predicate类型，也就是将传入的条件和当前条件以或(OR)的关系组合 

---
1. 加载路由中的Predicate
>在路由定位器中以及看到了通过路由定义转换路由方法，其中包含了通过谓语定义(PredicateDefinition)转换谓语(Predicate)的部分,在RouteDefinitionRouteLocator类中源码如下：

```java
	/**
	 * 返回组合的谓词
	 * @param routeDefinition
	 * @return
	 */
	private Predicate<ServerWebExchange> combinePredicates(RouteDefinition routeDefinition) {
		//获取RouteDefinition中的PredicateDefinition集合
		List<PredicateDefinition> predicates = routeDefinition.getPredicates();

		Predicate<ServerWebExchange> predicate = lookup(routeDefinition, predicates.get(0));

		for (PredicateDefinition andPredicate : predicates.subList(1, predicates.size())) {
			Predicate<ServerWebExchange> found = lookup(routeDefinition, andPredicate);
			//返回一个组合的谓词，表示该谓词与另一个谓词的短路逻辑AND
			predicate = predicate.and(found);
		}

		return predicate;
	}
	
		/**
	 * 获取一个谓语定义（PredicateDefinition）转换的谓语
	 * @param route
	 * @param predicate
	 * @return
	 */
	@SuppressWarnings("unchecked")
	private Predicate<ServerWebExchange> lookup(RouteDefinition route, PredicateDefinition predicate) {
		//获取谓语创建工厂
		RoutePredicateFactory<Object> factory = this.predicates.get(predicate.getName());
		if (factory == null) {
            throw new IllegalArgumentException("Unable to find RoutePredicateFactory with name " + predicate.getName());
		}
		//获取参数
		Map<String, String> args = predicate.getArgs();
		if (logger.isDebugEnabled()) {
			logger.debug("RouteDefinition " + route.getId() + " applying "
					+ args + " to " + predicate.getName());
		}

		//组装参数
        Map<String, Object> properties = factory.shortcutType().normalize(args, factory, this.parser, this.beanFactory);
        //构建创建谓语的配置信息
		Object config = factory.newConfig();
        ConfigurationUtils.bind(config, properties,
                factory.shortcutFieldPrefix(), predicate.getName(), validator);
        if (this.publisher != null) {
            this.publisher.publishEvent(new PredicateArgsEvent(this, route.getId(), properties));
        }
        //通过谓语工厂构建谓语
        return factory.apply(config);
	}
	
```
> 1. 获取路由定义（routeDefinition）所有的谓语定位（PredicateDefinition）
> 2. 以此根据谓语定义（PredicateDefinition）查找谓语对于的创建工厂（RoutePredicateFactory） 创建谓语
> 3. 通过 Predicate<T>接口 and方法合并谓语集合返回一个新的复合谓语

2. 使用路由Predicate判断路由是否可用
> 在Spring-Cloud-Gateway之请求处理流程里我们已经了解在handlerMapping中通过路由定位器获取所有路由，并过滤掉谓语判断失败的路由，最终获取满足条件的路由，下面再看下过滤的具体逻辑。

```java
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		//通过路由定位器获取路由信息
		return this.routeLocator.getRoutes()
				.filter(route -> {
			exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, route.getId());
					//返回通过谓语过滤的路由信息
					return route.getPredicate().test(exchange);
				})
	}
```
> 1. 通过routeLocator的getRoutes方法获取所有路由
> 2. 在所有路由中通过filter方法匹配Predicate.test匹配的路由信息

---
通过上面1，2两步我们看到了整个路由的条件创建以及使用的地方以及流程，Spring-Cloud-Gateway通过Predicate接口完成简单条件组合以及判断。
>接下来看通过RoutePredicateFactory创建PredicateSpring-Cloud-Gateway预制了很多RoutePredicateFactory使其可以通过简单的配置就可以创建出理想的Predicate

RoutePredicateFactory类图如下：
![image](https://raw.githubusercontent.com/guofazhan/image/master/RoutePredicateFactoryUML.png)
RoutePredicateFactory子类功能分类图：
![image](https://raw.githubusercontent.com/guofazhan/image/master/PredicateFactory.png)

---
- MethodRoutePredicateFactory

```java
/**
 * 请求方式（GET,POST,DEL,PUT）校验匹配创建工厂
 * @author Spencer Gibb
 */
public class MethodRoutePredicateFactory extends AbstractRoutePredicateFactory<MethodRoutePredicateFactory.Config> {

	public static final String METHOD_KEY = "method";
	@Override
	public Predicate<ServerWebExchange> apply(Config config) {
		return exchange -> {
			//获取当前请求的HttpMethod
			HttpMethod requestMethod = exchange.getRequest().getMethod();
			//校验请求HttpMethod与配置是否一致
			return requestMethod == config.getMethod();
		};
	}

	public static class Config {
		/**
		 * http 请求Method
		 */
		private HttpMethod method;

		public HttpMethod getMethod() {
			return method;
		}

		public void setMethod(HttpMethod method) {
			this.method = method;
		}
	}
}
```
 配置列子
 
```java
spring:
  cloud:
    gateway:
      routes:
      # =====================================
      - id: method_route
        uri: http://example.org
        predicates:
        - Method=GET
```
> 1. Method=GET 会被解析成PredicateDefinition对象 （name =Method ，args= GET）
> 2. 通过PredicateDefinition的Name找到MethodRoutePredicateFactory工厂
> 3. 通过 PredicateDefinition 的args 创建Config对象（HttpMethod=GET）
> 4. 通过 MethodRoutePredicateFactory工厂的apply方法传入config创建Predicate对象。

>在此只对MethodRoutePredicateFactory做了描述和说明，其它的谓语工程基本与此一致。此时Spring-Cloud-Gateway的RoutePredicate所有逻辑代码都已经阅读完毕，对阅读过程做文档记录以便加深印象，后期翻阅
