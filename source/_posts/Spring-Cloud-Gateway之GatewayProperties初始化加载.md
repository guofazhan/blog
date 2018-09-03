title: Spring-Cloud-Gateway之GatewayProperties初始化加载
author: Mr.G
tags:
  - 微服务
  - 网关
categories:
  - Spring-Cloud-Gateway
date: 2018-09-03 08:55:00
---
在Spring-Cloud-Gateway初始化时我们在GatewayAutoConfiguration配置中看到了有初始化加载GatewayProperties实例的配置，接下来学习下GatewayProperties都加载了哪些信息

---

<!-- more -->
GatewayAutoConfiguration中我看到加载GatewayProperties如下：

```
    //加载配置beans
	// ConfigurationProperty beans
	@Bean
	public GatewayProperties gatewayProperties() {
		return new GatewayProperties();
	}
```
> GatewayProperties的代码如下：

```
/**
 * 网关配置信息加载
 * 从appliccation.yml中解析前缀为spring.cloud.gateway的配置
 * @author Spencer Gibb
 */
@ConfigurationProperties("spring.cloud.gateway")
@Validated
public class GatewayProperties {

	/**
	 * 路由定义列表
	 * 加载配置key=spring.cloud.gateway.routes 列表
	 * List of Routes
	 */
	@NotNull
	@Valid
	private List<RouteDefinition> routes = new ArrayList<>();

	/**
	 * 默认的过滤器定义列表
	 * 加载配置 key = spring.cloud.gateway.default-filters 列表
	 * List of filter definitions that are applied to every route.
	 */
	private List<FilterDefinition> defaultFilters = new ArrayList<>();

	/**
	 * 网媒体类型列表
	 * 加载配置 key = spring.cloud.gateway.streamingMediaTypes 列表
	 * 默认包含{text/event-stream,application/stream+json}
	 */
	private List<MediaType> streamingMediaTypes = Arrays.asList(MediaType.TEXT_EVENT_STREAM,
			MediaType.APPLICATION_STREAM_JSON);
}			
```
由GatewayProperties代码可以看出其包含如下配置信息
> - spring.cloud.gateway.routes:网关路由定义配置，列表形式
> - spring.cloud.gateway.default-filters: 网关默认过滤器定义配置，列表形式
> - spring.cloud.gateway.streamingMediaTypes:网关网络媒体类型，列表形式

同时，我们看到代码中的routes其实RouteDefinition集合，defaultFilters是FilterDefinition集合，在Spring-Cloud-Gateway之route数据模型我们已经分析了这两个数据模型所有的字段含义以及类型了。通过数据模型以及代码可以看出配置中可以包含的具体信息。

---

- spring.cloud.gateway.routes
> - id:路由ID 编号，唯一
> - uri:  路由指向的URI
> - order: 顺序
> - predicates:谓语数组，列表形式

---

- spring.cloud.gateway.default-filters
> - name:过滤器定义名称
> - args:  参数

---
接下来看下Spring-Cloud-Gateway给的样例工程（spring-cloud-gateway-sample）中的配置

```
spring:
  cloud:
    gateway:
      default-filters:
      - PrefixPath=/httpbin
      - AddResponseHeader=X-Response-Default-Foo, Default-Bar
      routes:
      - id: websocket_test
        uri: ws://localhost:9000
        order: 9000
        predicates:
        - Path=/echo
      - id: default_path_to_httpbin
        uri: ${test.uri}
        order: 10000
        predicates:
        - Path=/**

```
- 备注
>  default-filters下配置PrefixPath=/httpbin字符串，可以查看FilterDefinition中构造函数，它接收一个text字符串解析字符传并创建实例信息。同样predicates配置与其一致。
字符传格式：name=param1,param2,param3.....。

如下：

```
	public FilterDefinition(String text) {
		int eqIdx = text.indexOf("=");
		if (eqIdx <= 0) {
			setName(text);
			return;
		}
		setName(text.substring(0, eqIdx));

		String[] args = tokenizeToStringArray(text.substring(eqIdx+1), ",");

		for (int i=0; i < args.length; i++) {
			this.args.put(NameUtils.generateName(i), args[i]);
		}
	}
```


上面的配置可以很容易的知道最终加载出来GatewayProperties实例中都包含哪些信息。
> 弄懂了配置的具体加载以及初始化会加深对网关整体流程的理解，合理的使用配置文件定义我们自己的需求，快速的根据配置定位，了解服务的详细情况




