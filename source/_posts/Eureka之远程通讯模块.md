title: Eureka之远程通讯模块
author: Mr.G
tags:
  - 微服务
  - Eureka
categories:
  - 微服务
date: 2018-08-25 01:10:00
---
> Eureka 架构为C-S模式 同时支持S 集群 ，不可避免的在client-server、server-server之间是需要远程通讯的，上面已经知道，Eureka 提供是http协议rest api，下面通过源码源码分析学习下eureka 提供的远程通讯模块

#### 提供远程通讯模块的包路径

``` java
com.netflix.discovery.shared.transport
```
#### 通讯模块对外的核心接口矩阵
![image](https://github.com/guofazhan/image/blob/master/%E9%80%9A%E8%AE%AF%E6%A0%B8%E5%BF%83%E6%8E%A5%E5%8F%A3.png?raw=true)
>  1. EurekaHttpClient 通讯请求接口，负责发送http请求
>  2. EurekaHttpClientFactory EurekaHttpClient创建工厂接口，负责创建高等级的client
>  3. TransportClientFactory EurekaHttpClient创建工厂接口，负责创建低等级的client
>  4. EurekaTransportConfig 通讯相关配置获取接口，负责在配置文件中读取通讯模块的配置信息

> **备注：** EurekaHttpClientFactory与TransportClientFactory同样都是创建client工厂，不过两个的职责不一样，client按功能划分为两大类，下面详细介绍。
<!-- more -->
#### EurekaHttpClient
![image](https://github.com/guofazhan/image/blob/master/EurekaHttpClient.png?raw=true)

> 通过上图可以明确的看出EurekaHttpClient接口实现为两大类:
> 1. 实现http通讯的lowlevel实现；
> 2. 使用装饰器模式实现特定功能的top level实现 。

> low level实现：  Eureka 根据http client 实现了两个AbstractJerseyEurekaHttpClient与AbstractJersey2EurekaHttpClient的low level 实现。从上面可以看出，如果Jersey1与Jersey2都不满足我们的自己的需求的话，我也可以根据自己需要的httpClient实现替代类，例如实现以OKhttp作为底层通讯的OkHttpEurekaHttpClient 或实现以netty作为底层通讯的NettyEurekaHttpClient，

> top level实现 ： 通过装饰器模式eueka 内置了 响应指标采集功能的装饰器、失败重试功能装饰器等，同时我们也可以扩展。

>** 备注：** HttpReplicationClient是eureka 专为server端集群节点通讯提供的通讯接口。其本质low level

- EurekaHttpClient接口方法

方法 | 描述
---|---
register(InstanceInfo info) | 服务注册
cancel(String appName, String id) |服务下线
sendHeartBeat| 服务续约
statusUpdate | 服务状态更新
deleteStatusOverride| 
getApplications(String... regions) | 获取服务注册列表
getDelta(String... regions) | 获取增量列表
getVip(String vipAddress, String... regions) |根据vip获取列表
getSecureVip(String secureVipAddress, String... regions) | 根据svip获取列表
getApplication(String appName)| 根据集群ID获取服务集群列表
getInstance(String appName, String id)| 根据集群ID与服务ID获取服务
getInstance(String id)|根据服务ID获取服务

- HttpReplicationClient接口方法 (server 节点数据同步特有)

方法 | 描述
---|---
statusUpdate(String asgName, ASGStatus newStatus) | 状态更新同步
submitBatchUpdates(ReplicationList replicationList)| 批量执行更新同步任务

#### EurekaTransportConfig通讯配置新获取

``` java
/**
 * eureka 远程通讯客户端配置默认实现
 * @author David Liu
 */
public class DefaultEurekaTransportConfig implements EurekaTransportConfig {
    private static final String SUB_NAMESPACE = TRANSPORT_CONFIG_SUB_NAMESPACE + ".";

    private final String namespace;
    private final DynamicPropertyFactory configInstance;

    public DefaultEurekaTransportConfig(String parentNamespace, DynamicPropertyFactory configInstance) {
        this.namespace = parentNamespace == null
                ? SUB_NAMESPACE
                : (parentNamespace.endsWith(".")
                    ? parentNamespace + SUB_NAMESPACE
                    : parentNamespace + "." + SUB_NAMESPACE);
        this.configInstance = configInstance;
    }

    @Override
    public int getSessionedClientReconnectIntervalSeconds() {
        return configInstance.getIntProperty(namespace + SESSION_RECONNECT_INTERVAL_KEY, Values.SESSION_RECONNECT_INTERVAL).get();
    }

```
> 1.构建函数创建 DynamicPropertyFactory
> 2. 构建函数 初始化namespace 配置key的前缀
> 3. 获取配置信息通过DynamicPropertyFactory 动态在配置文件中获取

#### EurekaHttpClients 工具类

``` java
public final class EurekaHttpClients {

    private static final Logger logger = LoggerFactory.getLogger(EurekaHttpClients.class);

    private EurekaHttpClients() {
    }

    public static EurekaHttpClientFactory queryClientFactory(ClusterResolver bootstrapResolver,
                                                             TransportClientFactory transportClientFactory,
                                                             EurekaClientConfig clientConfig,
                                                             EurekaTransportConfig transportConfig,
                                                             InstanceInfo myInstanceInfo,
                                                             ApplicationsResolver.ApplicationsSource applicationsSource) {

        ClosableResolver queryResolver = transportConfig.useBootstrapResolverForQuery()
                ? wrapClosable(bootstrapResolver)
                : queryClientResolver(bootstrapResolver, transportClientFactory,
                clientConfig, transportConfig, myInstanceInfo, applicationsSource);
        return canonicalClientFactory(EurekaClientNames.QUERY, transportConfig, queryResolver, transportClientFactory);
    }

    public static EurekaHttpClientFactory registrationClientFactory(ClusterResolver bootstrapResolver,
                                                                    TransportClientFactory transportClientFactory,
                                                                    EurekaTransportConfig transportConfig) {
        return canonicalClientFactory(EurekaClientNames.REGISTRATION, transportConfig, bootstrapResolver, transportClientFactory);
    }

    static EurekaHttpClientFactory canonicalClientFactory(final String name,
                                                          final EurekaTransportConfig transportConfig,
                                                          final ClusterResolver<EurekaEndpoint> clusterResolver,
                                                          final TransportClientFactory transportClientFactory) {

        return new EurekaHttpClientFactory() {
            @Override
            public EurekaHttpClient newClient() {
                return new SessionedEurekaHttpClient(
                        name,
                        RetryableEurekaHttpClient.createFactory(
                                name,
                                transportConfig,
                                clusterResolver,
                                RedirectingEurekaHttpClient.createFactory(transportClientFactory),
                                ServerStatusEvaluators.legacyEvaluator()),
                        transportConfig.getSessionedClientReconnectIntervalSeconds() * 1000
                );
            }

            @Override
            public void shutdown() {
                wrapClosable(clusterResolver).shutdown();
            }
        };
    }
```
> EurekaHttpClients，是创建EurekaHttpClientFactory的工具类

#### EurekaHttpClient Low Level部分
- JerseyEurekaHttpClientFactory

``` java
public class JerseyEurekaHttpClientFactory implements TransportClientFactory {

    public static final String HTTP_X_DISCOVERY_ALLOW_REDIRECT = "X-Discovery-AllowRedirect";

    private final EurekaJerseyClient jerseyClient;
    private final ApacheHttpClient4 apacheClient;
    private final ApacheHttpClientConnectionCleaner cleaner;
    private final Map<String, String> additionalHeaders;


    private JerseyEurekaHttpClientFactory(EurekaJerseyClient jerseyClient,
                                          ApacheHttpClient4 apacheClient,
                                          long connectionIdleTimeout,
                                          Map<String, String> additionalHeaders) {
        this.jerseyClient = jerseyClient;
        this.apacheClient = jerseyClient != null ? jerseyClient.getClient() : apacheClient;
        this.additionalHeaders = additionalHeaders;
        this.cleaner = new ApacheHttpClientConnectionCleaner(this.apacheClient, connectionIdleTimeout);
    }

    /**
     * 服务创建JerseyApplicationClient
     * @param endpoint
     * @return
     */
    @Override
    public EurekaHttpClient newClient(EurekaEndpoint endpoint) {
        return new JerseyApplicationClient(apacheClient, endpoint.getServiceUrl(), additionalHeaders);
    }
```


> JerseyEurekaHttpClientFactory 为TransportClientFactory的Jersey1实现主要负责创建JerseyEurekaHttpClient 


- AbstractJerseyEurekaHttpClient

``` java
public abstract class AbstractJerseyEurekaHttpClient implements EurekaHttpClient {

    private static final Logger logger = LoggerFactory.getLogger(AbstractJerseyEurekaHttpClient.class);

    /**
     * jersey1 http 客户端，负责底层发送http请求
     */
    protected final Client jerseyClient;
    /**
     * server url
     */
    protected final String serviceUrl;

    protected AbstractJerseyEurekaHttpClient(Client jerseyClient, String serviceUrl) {
        this.jerseyClient = jerseyClient;
        this.serviceUrl = serviceUrl;
        logger.debug("Created client for url: {}", serviceUrl);
    }

    /**
     * 服务注册
     * @param info
     * @return
     */
    @Override
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            //通过HTTP客户端发送http请求
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            //构建响应结果
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
```
> 1.通过构建函数 可以看到传入一个jersey1 http 客户端
> 2. register方法 实际是通过jersey1HttpClient发送http底层请求

> 在这里可以看到AbstractJerseyEurekaHttpClient实际上是不做底层通讯的工作的，全部都是交由com.sun.jersey.api.client.Client处理的，这个是在创建工厂创建时设置到AbstractJerseyEurekaHttpClient中的

> **备注:** AbstractJersey2EurekaHttpClient与AbstractJerseyEurekaHttpClient的实现原理基本一样只是底层通讯的client不一致，后边不在描述

#### EurekaHttpClient Top Level部分
- EurekaHttpClientDecorator

``` java
/**
 * 抽象Eureka远程通讯客户端装饰器，使用设计模式-装饰器模式 为客户端添加新的功能
 * @author Tomasz Bak
 */
public abstract class EurekaHttpClientDecorator implements EurekaHttpClient {

    /**
     * 请求类型
     */
    public enum RequestType {
        //注册
        Register,
        //下线
        Cancel,
        //心跳
        SendHeartBeat,
        //状态更新
        StatusUpdate,
        DeleteStatusOverride,
        GetApplications,
        GetDelta,
        GetVip,
        GetSecureVip,
        GetApplication,
        GetInstance,
        GetApplicationInstance
    }

    /**
     * 请求执行接口，负责执行请求
     * @param <R>
     */
    public interface RequestExecutor<R> {
        /**
         * 执行请求并返回响应信息
         * @param delegate 目标的客户端
         * @return
         */
        EurekaHttpResponse<R> execute(EurekaHttpClient delegate);

        /**
         * 请求的类型
         * @return
         */
        RequestType getRequestType();
    }

    /**
     * 抽象的执行方法，由子装饰器实现，附加其它功能
     * @param requestExecutor
     * @param <R>
     * @return
     */
    protected abstract <R> EurekaHttpResponse<R> execute(RequestExecutor<R> requestExecutor);

    @Override
    public EurekaHttpResponse<Void> register(final InstanceInfo info) {
        //创建一个注册请求的执行器，并执行
        return execute(new RequestExecutor<Void>() {
            @Override
            public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
                return delegate.register(info);
            }

            @Override
            public RequestType getRequestType() {
                return RequestType.Register;
            }
        });
    }

    @Override
    public EurekaHttpResponse<Void> cancel(final String appName, final String id) {
        //创建一个下线请求的执行，并执行
        return execute(new RequestExecutor<Void>() {
            @Override
            public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
                return delegate.cancel(appName, id);
            }

            @Override
            public RequestType getRequestType() {
                return RequestType.Cancel;
            }
        });
    }

```
> 基础的包装器 主要完成如下功能
> - 实现EurekaHttpClient接口每个基础方法，都有如下流程
> 1. 创建一个请求执行接口
> 2. 通过调用抽象的execute方法将RequestExecutor传入
> 3. 子类在RequestExecutor执行前或后完成新功能的附加

- EurekaHttpClientDecorator实现

实现类 |描述
---|---
MetricsCollectingEurekaHttpClient | 实现请求响应状态指标采集的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加响应状态指标采集向功能
RedirectingEurekaHttpClient | 实现请求重定向的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加请求重定向功能
RetryableEurekaHttpClient | 实现请求失败重试的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加失败重试功能
SessionedEurekaHttpClient | TODO


---
> 从以上来看eureka 通讯模块结构和功能还是非常明了清晰，同时后期如果需要扩展，也是非常方便与快捷的
