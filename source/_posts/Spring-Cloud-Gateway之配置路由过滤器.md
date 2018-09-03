title: Spring-Cloud-Gateway之配置路由过滤器
author: Mr.G
tags:
  - 微服务
  - 网关
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 17:01:00
---
上篇我们阅读了过滤器整个结构以及加载使用流程，接下来我们阅读下Spring-Cloud-Gateway对配置中的过滤器创建流程。

---
1. 加载GatewayFilter

在路由定位器中以及看到了通过路由定义转换路由方法，其中包含了通过过滤器定义(FilterDefinition)转换过滤器(GatewayFilter)的部分,在RouteDefinitionRouteLocator类中源码如下：

```
	/**
	 * 加载过滤器，根据过滤器的定义加载
	 * @param id
	 * @param filterDefinitions
	 * @return
	 */
	@SuppressWarnings("unchecked")
	private List<GatewayFilter> loadGatewayFilters(String id, List<FilterDefinition> filterDefinitions) {
		//遍历过滤器定义，将过滤器定义转换成对应的过滤器
		List<GatewayFilter> filters = filterDefinitions.stream()
				.map(definition -> {
					//通过过滤器定义名称获取过滤器创建工厂
					GatewayFilterFactory factory = this.gatewayFilterFactories.get(definition.getName());
					if (factory == null) {
                        throw new IllegalArgumentException("Unable to find GatewayFilterFactory with name " + definition.getName());
					}
					//获取参数
					Map<String, String> args = definition.getArgs();
					if (logger.isDebugEnabled()) {
						logger.debug("RouteDefinition " + id + " applying filter " + args + " to " + definition.getName());
					}

					//根据args组装配置信息
                    Map<String, Object> properties = factory.shortcutType().normalize(args, factory, this.parser, this.beanFactory);
					//构建过滤器创建配置信息
                    Object configuration = factory.newConfig();
                    ConfigurationUtils.bind(configuration, properties,
                            factory.shortcutFieldPrefix(), definition.getName(), validator);

                    //通过过滤器工厂创建GatewayFilter
                    GatewayFilter gatewayFilter = factory.apply(configuration);
                    if (this.publisher != null) {
                    	//发布事件
                        this.publisher.publishEvent(new FilterArgsEvent(this, id, properties));
                    }
                    return gatewayFilter;
				})
				.collect(Collectors.toList());

		ArrayList<GatewayFilter> ordered = new ArrayList<>(filters.size());
		//包装过滤器使其所有过滤器继承Ordered属性，可进行排序
		for (int i = 0; i < filters.size(); i++) {
			GatewayFilter gatewayFilter = filters.get(i);
			if (gatewayFilter instanceof Ordered) {
				ordered.add(gatewayFilter);
			}
			else {
				ordered.add(new OrderedGatewayFilter(gatewayFilter, i + 1));
			}
		}

		return ordered;
	}

	/**
	 * 获取RouteDefinition中的过滤器集合
	 * @param routeDefinition
	 * @return
	 */
	private List<GatewayFilter> getFilters(RouteDefinition routeDefinition) {
		List<GatewayFilter> filters = new ArrayList<>();

		//校验gatewayProperties是否含义默认的过滤器集合
		//TODO: support option to apply defaults after route specific filters?
		if (!this.gatewayProperties.getDefaultFilters().isEmpty()) {
			//加载全局配置的默认过滤器集合
			filters.addAll(loadGatewayFilters("defaultFilters",
					this.gatewayProperties.getDefaultFilters()));
		}

		if (!routeDefinition.getFilters().isEmpty()) {
			//加载路由定义中的过滤器集合
			filters.addAll(loadGatewayFilters(routeDefinition.getId(), routeDefinition.getFilters()));
		}

		//排序
		AnnotationAwareOrderComparator.sort(filters);
		return filters;
	}
```
> - getFilters方法 合并配置中的全局过滤器与路由自身配置的过滤器，并对其排序（全局配置过滤器信息通过gatewayProperties.getDefaultFilters()获取）

> - loadGatewayFilters 依次遍历路由定义下的FilterDefinition并将其通过对应的GatewayFilterFactory转换为GatewayFilter对象。

2. GatewayFilterFactory配置过滤器创建工厂创建GatewayFilter对象
> Spring-Cloud-Gateway默认内置很多GatewayFilterFactory实现类，用于创建作用不同的网关过滤器，下面通过图展示各个工厂类创建的过滤器的作用。

![image](https://raw.githubusercontent.com/guofazhan/image/master/GatewayFilterFactory.png)

 
- AddResponseHeaderGatewayFilterFactory 创建解析
 
```
/**
 *
 * 响应header添加数据过滤器
 * 用户在response header中添加配置数据
 * @author Spencer Gibb
 */
public class AddResponseHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {

	@Override
	public GatewayFilter apply(NameValueConfig config) {
		return (exchange, chain) -> {
			//获取Response并将配置的数据添加到header中
			exchange.getResponse().getHeaders().add(config.getName(), config.getValue());

			return chain.filter(exchange);
		};
	}
}
```
配置样例：

```
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Foo, Default-Bar
```
> 1. AddResponseHeader=X-Response-Default-Foo, Default-Bar 会被解析成FilterDefinition对象 （name =AddResponseHeader ，args= [X-Response-Default-Foo,Default-Bar]）
> 2. 通FilterDefinition的Name找到AddResponseHeaderGatewayFilterFactory工厂
> 3. 通过FilterDefinition 的args 创建Config对象（name=X-Response-Default-Foo,value=Default-Bar）
> 4. 通过 AddResponseHeaderGatewayFilterFactory工厂的apply方法传入config创建GatewayFilter对象。
