title: Eureka之服务发现LookupService
author: Mr.G
tags:
  - 微服务
  - Eureka
categories:
  - 微服务
date: 2018-08-28 11:38:00
---
> Eureka 注册中心，核心为提供服务注册，以及服务发现功能，下面我们重点讨论下Eureka核心功能之一的服务发现。
我们知道Eureka采用的C-S架构，客户端要通过请求服务端获取注册列表，提供给服务使用，服务端要保存客户端发来的注册信息保存下（Eureka采用的是内存存储），并提供给资源服务Resources使用，有关资源服务可见Eureka之REST API
<!-- more -->
#### LookupService体系图
![image](https://github.com/guofazhan/image/blob/master/LookupService.png?raw=true)
> 1. InstanceRegistry为Server端对资源服务提供服务发现功能的接口，用来读取服务端本地缓存注册表信息
> 2. EurekaClient 为Client端实现的服务发现接口，用来获取客户端本地缓存中的注册表信息
> 3. RemoteRegionRegistry 为Server用来读取其它区域的服务注册表的接口，并将其它区域的服务注册表缓存到本地，供本地InstanceRegistry使用。

#### Eureka区域，分区
>  eureka提供了region和zone两个概念来进行分区，这两个概念均来自于亚马逊的AWS：
> 1. region：可以简单理解为地理上的分区，比如亚洲地区，或者华北地区，再或者北京等等，没有具体大小的限制。根据项目具体的情况，可以自行合理划分region。RemoteRegionRegistry就是其它分区的服务注册信息。
> 2. zone：可以简单理解为region内的具体机房，比如说region划分为北京，然后北京有两个机房，就可以在此region之下划分出zone1,zone2两个zone。

> RemoteRegionRegistry 根据配置实例化：         
> 1. 配置KEY： eureka.remoteRegionUrlsWithName   
> 2. 格式： eureka.remoteRegionUrlsWithName=region1;http://region1host/eureka/v2,region2;http://region2host/eureka/v2
> 3. 解析: regionName->remoteRegionURL 根据解析出来的MAP 创建相应的 RemoteRegionRegistry

> ***备注：*** 一般中小行项目基本所有服务都部署在同一机房，不存在区域划分这些，即初始化时:RemoteRegionRegistry List为空,具体可参见AbstractInstanceRegistry的initRemoteRegionRegistry方法
#### LookupService类图以及实现
![image](https://github.com/guofazhan/image/blob/master/LookupServiceUml.png?raw=true)
##### 服务发现LookupService接口

```java
/**
 * 服务发现接口
 * Lookup service for finding active instances.
 *
 * @author Karthik Ranganathan, Greg Kim.
 * @param <T> for backward compatibility

 */
public interface LookupService<T> {

    /**
     * 根据集群ID获取集群列表
     * Returns the corresponding {@link Application} object which is basically a
     * container of all registered <code>appName</code> {@link InstanceInfo}s.
     *
     * @param appName
     * @return a {@link Application} or null if we couldn't locate any app of
     *         the requested appName
     */
    Application getApplication(String appName);

    /**
     * 获取服务注册列表
     * Returns the {@link Applications} object which is basically a container of
     * all currently registered {@link Application}s.
     *
     * @return {@link Applications}
     */
    Applications getApplications();

    /**
     * 根据实例ID获取实例信息
     */
    List<InstanceInfo> getInstancesById(String id);

    /**
     * 轮询获取集群中的下一个可用的注册服务，起到负载均衡作用，保证每个可用注册服务均衡的被使用，此方法仅提供个Eureka 客户端使用实现，服务端默认为空实现
     * Gets the next possible server to process the requests from the registry
     * information received from eureka.
     */
    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);
```
> 从接口可以看到提供了四个功能方法如下：
> 1. getApplication:根据集群ID获取集群列表
> 2. getApplications:获取服务注册列表
> 3. getInstancesById:根据实例ID获取实例信息
> 4. getNextServerFromEureka:轮询获取集群中的下一个可用的注册服务，起到负载均衡作用，保证每个可用注册服务均衡的被使用，此方法仅提供个Eureka 客户端使用实现，服务端默认为空实现

##### 服务端实现AbstractInstanceRegistry

1. 缓存服务注册属性

```java

   /**
     * 服务注册表
     */
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```
2. 其它区域服务发现集合属性

```java
    /**
     * 远程区域服务发现注册服务集合
     */
    protected Map<String, RemoteRegionRegistry> regionNameVSRemoteRegistry = new HashMap<String, RemoteRegionRegistry>();
```
3. getApplications功能方法实现

```java
    /**
     * 获取服务注册列表信息
     * Get all applications in this instance registry, falling back to other regions if allowed in the Eureka config.
     *
     * @return the list of all known applications
     *
     * @see com.netflix.discovery.shared.LookupService#getApplications()
     */
    public Applications getApplications() {
        //配置参数，是否禁用远程区域的服务注册列表
        boolean disableTransparentFallback = serverConfig.disableTransparentFallbackToOtherRegion();
        if (disableTransparentFallback) {
            //获取本地服务注册列表
            return getApplicationsFromLocalRegionOnly();
        } else {
            //获取服务注册列表在所有区域中，包含本地
            return getApplicationsFromAllRemoteRegions();  // Behavior of falling back to remote region can be disabled.
        }
    }
    
        /**
     * 获服务注册列表,在所有区域
     * Returns applications including instances from all remote regions. <br/>
     * Same as calling {@link #getApplicationsFromMultipleRegions(String[])} with a <code>null</code> argument.
     */
    public Applications getApplicationsFromAllRemoteRegions() {
        //根据区域信息获取服务注册列表，此处区域列表传入所有远程区域列表
        return getApplicationsFromMultipleRegions(allKnownRemoteRegions);
    }

    /**
     * 获取本地服务注册列表
     * Returns applications including instances from local region only. <br/>
     * Same as calling {@link #getApplicationsFromMultipleRegions(String[])} with an empty array.
     */
    @Override
    public Applications getApplicationsFromLocalRegionOnly() {
        //根据区域信息获取服务注册列表，此处区域列表传入空值
        return getApplicationsFromMultipleRegions(EMPTY_STR_ARRAY);
    }
    
        /**
     * 根据区域信息查询服务注册列表
     */
    public Applications getApplicationsFromMultipleRegions(String[] remoteRegions) {

        //是否根据远程区域查询服务注册列表标识
        boolean includeRemoteRegion = null != remoteRegions && remoteRegions.length != 0;

        logger.debug("Fetching applications registry with remote regions: {}, Regions argument {}",
                includeRemoteRegion, Arrays.toString(remoteRegions));

        //记录统计监控数据
        if (includeRemoteRegion) {
            GET_ALL_WITH_REMOTE_REGIONS_CACHE_MISS.increment();
        } else {
            GET_ALL_CACHE_MISS.increment();
        }

        //构建服务注册列表实例信息
        Applications apps = new Applications();
        apps.setVersion(1L);
        //加入本地注册信息到服务注册列表实例中
        for (Entry<String, Map<String, Lease<InstanceInfo>>> entry : registry.entrySet()) {
            Application app = null;

            if (entry.getValue() != null) {
                for (Entry<String, Lease<InstanceInfo>> stringLeaseEntry : entry.getValue().entrySet()) {
                    Lease<InstanceInfo> lease = stringLeaseEntry.getValue();
                    if (app == null) {
                        app = new Application(lease.getHolder().getAppName());
                    }
                    app.addInstance(decorateInstanceInfo(lease));
                }
            }
            if (app != null) {
                apps.addApplication(app);
            }
        }
        //是否加入远程区域的注册信息
        if (includeRemoteRegion) {
            //遍历远程区域，添加远程区域的注册信息到服务注册列表实例
            for (String remoteRegion : remoteRegions) {
                RemoteRegionRegistry remoteRegistry = regionNameVSRemoteRegistry.get(remoteRegion);
                if (null != remoteRegistry) {
                    Applications remoteApps = remoteRegistry.getApplications();
                    for (Application application : remoteApps.getRegisteredApplications()) {
                        if (shouldFetchFromRemoteRegistry(application.getName(), remoteRegion)) {
                            logger.info("Application {}  fetched from the remote region {}",
                                    application.getName(), remoteRegion);

                            Application appInstanceTillNow = apps.getRegisteredApplications(application.getName());
                            if (appInstanceTillNow == null) {
                                appInstanceTillNow = new Application(application.getName());
                                apps.addApplication(appInstanceTillNow);
                            }
                            for (InstanceInfo instanceInfo : application.getInstances()) {
                                appInstanceTillNow.addInstance(instanceInfo);
                            }
                        } else {
                            logger.debug("Application {} not fetched from the remote region {} as there exists a "
                                            + "whitelist and this app is not in the whitelist.",
                                    application.getName(), remoteRegion);
                        }
                    }
                } else {
                    logger.warn("No remote registry available for the remote region {}", remoteRegion);
                }
            }
        }
        //设置实例的hashCode
        apps.setAppsHashCode(apps.getReconcileHashCode());
        return apps;
    }
```
> getApplications功能主要由以下方法完成：
> 1. getApplications : 此方法根据配置参数信息，判断是只从本地缓存中获取注册列表(getApplicationsFromLocalRegionOnly)还是从所以区域获取注册列表(getApplicationsFromAllRemoteRegions)，参数信息：disableTransparentFallback
> 2. getApplicationsFromLocalRegionOnly :获取本地缓存中的注册表信息，调用getApplicationsFromMultipleRegions传入空区域集合
> 3. getApplicationsFromAllRemoteRegions : 从所有区域查找注册表信息，传入全部区域集合。
> 4. getApplicationsFromMultipleRegions ：根据传入区域信息查询注册表信息

>  getApplicationsFromMultipleRegions完成如下功能:
>  1. 构建空注册表实例：Applications
>  2. 添加本地缓存的注册信息到Applications
>  3. 判断是否查找其它区域
>  4. 遍历其它区域RemoteRegionRegistry获取Applications
>  5. 添加其它区域中注册的信息到Applications（剔除重复的）
>  6. 设置apps的hashCode 用于增量更新

4. getApplications功能方法实现

```java
    /**
     * 根据服务集群ID，查询服务集群列表
     *
     * @param appName the application name of the application
     * @return the application
     *
     * @see com.netflix.discovery.shared.LookupService#getApplication(java.lang.String)
     */
    @Override
    public Application getApplication(String appName) {
        //获取配置信息
        //配置参数，是否禁用远程区域的服务注册列表
        boolean disableTransparentFallback = serverConfig.disableTransparentFallbackToOtherRegion();
        return this.getApplication(appName, !disableTransparentFallback);
    }


    /**
     * 根据服务集群ID，查询服务集群列表
     * Get application information.
     * @return the application
     */
    @Override
    public Application getApplication(String appName, boolean includeRemoteRegion) {
        Application app = null;

        //获取本地注册列表中的服务集群信息
        Map<String, Lease<InstanceInfo>> leaseMap = registry.get(appName);

        if (leaseMap != null && leaseMap.size() > 0) {
            //构建集群实例Application
            for (Entry<String, Lease<InstanceInfo>> entry : leaseMap.entrySet()) {
                if (app == null) {
                    app = new Application(appName);
                }
                app.addInstance(decorateInstanceInfo(entry.getValue()));
            }
        } else if (includeRemoteRegion) {
            //包含远程区域时，通过远程区域注册服务查询集群信息
            for (RemoteRegionRegistry remoteRegistry : this.regionNameVSRemoteRegistry.values()) {
                Application application = remoteRegistry.getApplication(appName);
                if (application != null) {
                    return application;
                }
            }
        }
        //返回集群实例
        return app;
    }
```
> getApplication功能主要由以下方法完成：
> getApplication ： 根据 集群名称以及参数是否禁用其它区域来获取集群列表Application
> 1. 通过本地缓存注册信息获取组装集群实例Application
> 2. 当本地缓存中未查到集群信息且可用远程区域发现服务查询时，遍历远程区域发现服务查询集群信息组装
集群实例Application
> 3. 返回集群实例Application

5. getInstancesById功能方法实现
> 与4功能类似
6. getNextServerFromEureka功能方法实现
> 空实现

##### 客户端实现DiscoveryClient


```java
    /*
     * (non-Javadoc)
     * @see com.netflix.discovery.shared.LookupService#getApplication(java.lang.String)
     */
    @Override
    public Application getApplication(String appName) {
        return getApplications().getRegisteredApplications(appName);
    }

    /*
     * (non-Javadoc)
     *
     * @see com.netflix.discovery.shared.LookupService#getApplications()
     */
    @Override
    public Applications getApplications() {
        return localRegionApps.get();
    }
    
        /*
     * (non-Javadoc)
     * @see com.netflix.discovery.shared.LookupService#getInstancesById(java.lang.String)
     */
    @Override
    public List<InstanceInfo> getInstancesById(String id) {
        List<InstanceInfo> instancesList = new ArrayList<InstanceInfo>();
        for (Application app : this.getApplications()
                .getRegisteredApplications()) {
            InstanceInfo instanceInfo = app.getByInstanceId(id);
            if (instanceInfo != null) {
                instancesList.add(instanceInfo);
            }
        }
        return instancesList;
    }
    
```
> 在客户端的实现方法中可以清晰的看到是通过localRegionApps.get()获取服务注册列表实例，此实例是在本地缓存存储且为原子性的。下面我们主要查看localRegionApps是怎么加载以及更新

 - 构造函数首次加载localRegionApps
 
```java

        //通过fetchRegistry方法首次从远程Server端获取注册列表缓存到客户端本地，默认选择增量拉取
        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }
```
> 在构造函数中调用fetchRegistry 首次初始化从服务端拉取服务注册实例并缓存到客户端内存中
- 初始化定时任务，定时拉取服务端注册信息刷新本地缓存

```
    /**
     * Initializes all scheduled tasks.
     */
    private void initScheduledTasks() {
        //判断客户端远程拉取注册表禁用配置，true，标识客户端运行定时拉取远程注册表配置
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            //定时任务定时执行客户端远程拉取注册表来更新客户端本地缓存
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
    }
    
        /**
     * 远程拉取Server端注册表信息更新客户端本地缓存，定时任务，客户端定时执行此任务
     * The task that fetches the registry information at specified intervals.
     *
     */
    class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }
    
```
> 1. 构造函数初始化定时任务，调用initScheduledTasks
> 2. 定时任务，初始化刷新服务注册表实例定时任务
> 3. CacheRefreshThread线程定时执行，调用refreshRegistry()方法从server端拉取注册信息并刷新本地缓存

- fetchRegistry功能

```java
/**
     * 客户端拉取Eureka Server服务注册列表，本地缓存 (forceFullRegistryFetch 是否全量拉取标识)
     */
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            //本地缓存中获取注册列表信息
            Applications applications = getApplications();

            //判断全量拉取还是增量拉取
            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                //全量拉取注册表信息
                getAndStoreFullRegistry();
            } else {
                //增量拉取注册表信息
                getAndUpdateDelta(applications);
            }
            //设置hashCode，注册列表判断变化是根据hashCode
            applications.setAppsHashCode(applications.getReconcileHashCode());
            //日志打印统计
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + appPathIdentifier + " - was unable to refresh its cache! status = " + e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        //通知缓存刷新事件
        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        //更新实例的远程状态
        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
    
       /**
     * 全量拉取Eureka Server端的注册列表信息
     */
    private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        //通过远程通讯组件请求Server端获取全量注册列表
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {
            logger.error("The application is null for some reason. Not storing this information");
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            //获取的注册列表缓存到本地
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
    
        /**
     * 增量拉取Eureka Server端的注册列表信息
     */
    private void getAndUpdateDelta(Applications applications) throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        Applications delta = null;
        //通过远程通讯组件请求Server端获取增量注册列表
        EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            delta = httpResponse.getEntity();
        }

        if (delta == null) {
            logger.warn("The server does not allow the delta revision to be applied because it is not safe. "
                    + "Hence got the full registry.");
            //增量获取为空时，全量拉取
            getAndStoreFullRegistry();
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
            String reconcileHashCode = "";
            //拉取注册表更新锁
            if (fetchRegistryUpdateLock.tryLock()) {
                try {
                    //更新增量信息到本地缓存中的注册表中
                    updateDelta(delta);
                    //获取更新后的HashCode
                    reconcileHashCode = getReconcileHashCode(applications);
                } finally {
                    fetchRegistryUpdateLock.unlock();
                }
            } else {
                logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
            }
            //判断本地的hashCode是否与远程hashCode一致，不一致时，需要同步一致
            // There is a diff in number of instances for some reason
            if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
                reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
            }
        } else {
            logger.warn("Not updating application delta as another thread is updating it already");
            logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
        }
    }
```
> 1. fetchRegistry : 根据传入参数以及配置信息判断是否全量拉取还是增量拉取，全量拉取调用getAndStoreFullRegistry方法 ；增量拉取调用getAndUpdateDelta方法
> 2. getAndStoreFullRegistry：全量从Server拉取服务注册信息，更新本地缓存。此方法通过Eureka提供的远程通讯模块调用Server端暴露的Resources信息获取服务注册列表实例apps
> 3. getAndUpdateDelta ：增量从Server端拉取服务的增量注册信息，更新本地缓存实例。


##### 远程区域服务发现实现RemoteRegionRegistry
> 与客户端实现方式基本一致，不在描述

#### 客户端，服务端，远程区域服务端交互
![image](https://github.com/guofazhan/image/blob/master/LookupService1.png?raw=true)



---
