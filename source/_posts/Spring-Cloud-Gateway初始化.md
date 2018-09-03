title: Spring-Cloud-Gateway初始化
author: Mr.G
tags:
  - 微服务
  - 网关
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 16:49:00
---
Spring-Cloud项目使用EnableAutoConfiguration注解自动
初始化配置信息，Spring-Cloud-Gateway同样，Spring-Cloud-Gateway下的spring.factories如下:

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
//依赖包的校验配置
org.springframework.cloud.gateway.config.GatewayClassPathWarningAutoConfiguration,\
//网关的核心配置
org.springframework.cloud.gateway.config.GatewayAutoConfiguration,\
//负载均衡相关依赖配置信息
org.springframework.cloud.gateway.config.GatewayLoadBalancerClientAutoConfiguration,\
//流控的依赖配置信息
org.springframework.cloud.gateway.config.GatewayRedisAutoConfiguration,\
//注册中心相关的依赖配置
org.springframework.cloud.gateway.discovery.GatewayDiscoveryClientAutoConfiguration

```
<!-- more -->
#### 析配置功能
>   分析配置功能前先了解下springboot常用的注解的含义

```
执行顺序
@AutoConfigureAfter：在指定的配置类初始化后再加载
@AutoConfigureBefore：在指定的配置类初始化前加载
@AutoConfigureOrder：数越小越先初始化

```

```
条件配置
@ConditionalOnClass ： classpath中存在该类时起效
@ConditionalOnMissingClass ： classpath中不存在该类时起效
@ConditionalOnBean ： DI容器中存在该类型Bean时起效
@ConditionalOnMissingBean ： DI容器中不存在该类型Bean时起效
@ConditionalOnSingleCandidate ： DI容器中该类型Bean只有一个或@Primary的只有一个时起效
@ConditionalOnExpression ： SpEL表达式结果为true时
@ConditionalOnProperty ： 参数设置或者值一致时起效
@ConditionalOnResource ： 指定的文件存在时起效
@ConditionalOnJndi ： 指定的JNDI存在时起效
@ConditionalOnJava ： 指定的Java版本存在时起效
@ConditionalOnWebApplication ： Web应用环境下起效
@ConditionalOnNotWebApplication ： 非Web应用环境下起效
```
---
1.  GatewayClassPathWarningAutoConfiguration配置
> GatewayClassPathWarningAutoConfiguration配置用于检查项目是否正确导入 spring-boot-starter-webflux 依赖，而不是错误导入 spring-boot-starter-web 依赖，同时 GatewayClassPathWarningAutoConfiguration配置在EnableAutoConfiguration配置加载前加载
接下来我看代码如何实现

```
@Configuration
//执行顺序注解
//当前注解标识需要在GatewayAutoConfiguration前加载此配置
@AutoConfigureBefore(GatewayAutoConfiguration.class)
public class GatewayClassPathWarningAutoConfiguration {
	@Configuration
	//条件判断注解
	//classpath中存在org.springframework.web.servlet.DispatcherServlet时起效，标识项目导入了spring-boot-starter-web包
	@ConditionalOnClass(name = "org.springframework.web.servlet.DispatcherServlet")
	protected static class SpringMvcFoundOnClasspathConfiguration {

		public SpringMvcFoundOnClasspathConfiguration() {
		   //当前项目导入了spring-boot-starter-web依赖时，打印警告日志
			log.warn(BORDER+"Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway at this time. "+
					"Please remove spring-boot-starter-web dependency."+BORDER);
		}

	}
	@Configuration
	//条件判断注解
	//classpath中不存在org.springframework.web.reactive.DispatcherHandler时起效，标识项目未导入了spring-boot-starter-webflux包
	@ConditionalOnMissingClass("org.springframework.web.reactive.DispatcherHandler")
	protected static class WebfluxMissingFromClasspathConfiguration {
		public WebfluxMissingFromClasspathConfiguration() {
	  //当前项目未导入了boot-starter-webflux依赖时，打印警告日志
			log.warn(BORDER+"Spring Webflux is missing from the classpath, which is required for Spring Cloud Gateway at this time. "+
					"Please add spring-boot-starter-webflux dependency."+BORDER);
		}

	}
}
```
2. GatewayLoadBalancerClientAutoConfiguration
> GatewayLoadBalancerClientAutoConfiguration配置作用是初始化 LoadBalancerClientFilter 路由的负载均衡拦截器

```
@Configuration
//条件判断注解
//classpath中存在LoadBalancerClient和RibbonAutoConfiguration和DispatcherHandler时此配置起效
@ConditionalOnClass({LoadBalancerClient.class, RibbonAutoConfiguration.class, DispatcherHandler.class})
//执行顺序注解
@AutoConfigureAfter(RibbonAutoConfiguration.class)
public class GatewayLoadBalancerClientAutoConfiguration {
	// GlobalFilter beans
	@Bean
	//条件判断注解
	//DI容器中存在LoadBalancerClient类型Bean时起效
	@ConditionalOnBean(LoadBalancerClient.class)
	public LoadBalancerClientFilter loadBalancerClientFilter(LoadBalancerClient client) {
		return new LoadBalancerClientFilter(client);
	}
}
```
3. GatewayRedisAutoConfiguration
> GatewayRedisAutoConfiguration 配置作用是初始化初始化 RedisRateLimiter  限流功能的RequestRateLimiterGatewayFilterFactory 基于 RedisRateLimiter 实现网关的限流功能，**代码略**
4. GatewayAutoConfiguration
> GatewayAutoConfiguration配置是Spring Cloud Gateway 核心配置类，初始化如下 ： 
>  - NettyConfiguration 底层通信netty配置
>  - GlobalFilter  （AdaptCachedBodyGlobalFilter，RouteToRequestUrlFilter，ForwardRoutingFilter，ForwardPathFilter，WebsocketRoutingFilter，WeightCalculatorWebFilter等）
>  - FilteringWebHandler
>  - GatewayProperties
>  - PrefixPathGatewayFilterFactory
>  - RoutePredicateFactory
>  - RouteDefinitionLocator
>  - RouteLocator
>  - RoutePredicateHandlerMapping 查找匹配到 Route并进行处理
>  - GatewayWebfluxEndpoint 管理网关的 HTTP API 

```
@Configuration
//条件注解
//通过 spring.cloud.gateway.enabled配置网关的开启与关闭
//matchIfMissing = true => 网关默认开启。
@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)
@EnableConfigurationProperties
@AutoConfigureBefore(HttpHandlerAutoConfiguration.class)
@AutoConfigureAfter({GatewayLoadBalancerClientAutoConfiguration.class, GatewayClassPathWarningAutoConfiguration.class})
@ConditionalOnClass(DispatcherHandler.class)
public class GatewayAutoConfiguration {
   	@Configuration
   	//当classpath中存在HttpClient起效
	@ConditionalOnClass(HttpClient.class)
	protected static class NettyConfiguration {
	    .....略
	}
	
	//加载配置beans
	// ConfigurationProperty beans
	@Bean
	public GatewayProperties gatewayProperties() {
		return new GatewayProperties();
	}
	@Bean
	public SecureHeadersProperties secureHeadersProperties() {
		return new SecureHeadersProperties();
	}
	
	// GlobalFilter beans
    //加载全局的拦截器
	@Bean
	public AdaptCachedBodyGlobalFilter adaptCachedBodyGlobalFilter() {
		return new AdaptCachedBodyGlobalFilter();
	}

	@Bean
	public RouteToRequestUrlFilter routeToRequestUrlFilter() {
		return new RouteToRequestUrlFilter();
	}

	@Bean
	@ConditionalOnBean(DispatcherHandler.class)
	public ForwardRoutingFilter forwardRoutingFilter(DispatcherHandler dispatcherHandler) {
		return new ForwardRoutingFilter(dispatcherHandler);
	}

	@Bean
	public ForwardPathFilter forwardPathFilter() {
		return new ForwardPathFilter();
	}

	@Bean
	public WebSocketService webSocketService() {
		return new HandshakeWebSocketService();
	}
	@Bean
	public WebsocketRoutingFilter websocketRoutingFilter(WebSocketClient webSocketClient,
		return new WebsocketRoutingFilter(webSocketClient, webSocketService, headersFilters);
	}
	@Bean
	public WeightCalculatorWebFilter weightCalculatorWebFilter(Validator validator) {
		return new WeightCalculatorWebFilter(validator);
	}
}
```
5. GatewayDiscoveryClientAutoConfiguration
> GatewayDiscoveryClientAutoConfiguration配置的作用是初始化配置路由中的注册发现服务信息

```
@Configuration
//同4
@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)
@AutoConfigureBefore(GatewayAutoConfiguration.class)
@ConditionalOnClass({DispatcherHandler.class, DiscoveryClient.class})
@EnableConfigurationProperties
public class GatewayDiscoveryClientAutoConfiguration {

	@Bean
	//当classpath中存在DiscoveryClient起效
	@ConditionalOnBean(DiscoveryClient.class)
	//通过spring.cloud.gateway.discovery.locator.enabled配置注册中心查找的开启与关闭
	@ConditionalOnProperty(name = "spring.cloud.gateway.discovery.locator.enabled")
	public DiscoveryClientRouteDefinitionLocator discoveryClientRouteDefinitionLocator(
			DiscoveryClient discoveryClient, DiscoveryLocatorProperties properties) {
		return new DiscoveryClientRouteDefinitionLocator(discoveryClient, properties);
	}
}
```
**通过自动加载初始化上述五个配置实例Spring-Cloud-Gateway就完成自身的信息加载和初始化工作，而对于使用者来说是无感知的**
 


