title: Eureka Server之注册服务实例租约信息管理LeaseManager
author: Mr.G
tags:
  - Eureka
  - 微服务
categories:
  - 微服务
date: 2018-09-03 16:41:00
---
> Eureka 注册中心服务端对注册进来的服务信息提供了一个基于时间的可用性管理(租约Lease)。通过此来管理注册服务的有效性
<!-- more -->

#### 提供租约管理的包路径

```
com.netflix.eureka.lease
```

#### 描述注册服务基于时间可用性

```java
/**
 * 实例租约包装类，描述实例基于时间的可用性
 */
public class Lease<T> {

    /**
     * 实例针对租约变化的动作类型
     */
    enum Action {
        //实例注册
        Register,
        //实例取消
        Cancel,
        //实例刷新
        Renew
    };

    /**
     * 默认的租约有效时长 90s
     */
    public static final int DEFAULT_DURATION_IN_SECS = 90;

    /**
     * 实例信息，此处默认为 InstanceInfo
     */
    private T holder;
    /**
     * 取消注册时间
     */
    private long evictionTimestamp;
    /**
     * 注册时间
     */
    private long registrationTimestamp;
    /**
     *
     */
    private long serviceUpTimestamp;
    /**
     * 最后更新时间
     */
    // Make it volatile so that the expiration task would see this quicker
    private volatile long lastUpdateTimestamp;

    /**
     * 租约的有效时长
     */
    private long duration;

    /**
     * 构造函数
     * @param r
     * @param durationInSecs
     */
    public Lease(T r, int durationInSecs) {
        holder = r;
        registrationTimestamp = System.currentTimeMillis();
        lastUpdateTimestamp = registrationTimestamp;
        duration = (durationInSecs * 1000);

    }
```
> 在Lease类中可以看到是使用了装饰模式对注册实例进行了包装，提供了基于时间可用性的新功能。
> - enum Action  定义了租约时间变更的动作类型，此处包含注册，下线，刷新（心跳发送刷新租约信息）等三个动作
> - evictionTimestamp：服务下线时间
> - registrationTimestamp: 服务注册时间
> - lastUpdateTimestamp: 最后更新时间
> - duration : 租约的有效时长默认90s

> Lease类对实例InstanceInfo进行功能扩展的方法：

1. renew
```java
    /**
     * 刷新租约的时间信息，租约续订更新最后更新时间为当前时间+租约有效时长
     * Renew the lease, use renewal duration if it was specified by the
     * associated {@link T} during registration, otherwise default duration is
     * {@link #DEFAULT_DURATION_IN_SECS}.
     */
    public void renew() {
        lastUpdateTimestamp = System.currentTimeMillis() + duration;

    }
```
2. cancel

```java
    /**
     * 注册列表剔除当前服务，更新租约的时间信息
     * Cancels the lease by updating the eviction time.
     */
    public void cancel() {
        if (evictionTimestamp <= 0) {
            evictionTimestamp = System.currentTimeMillis();
        }
    }
```

3. serviceUp

```java
    /**
     * 服务上线时间
     * Mark the service as up. This will only take affect the first time called,
     * subsequent calls will be ignored.
     */
    public void serviceUp() {
        if (serviceUpTimestamp == 0) {
            serviceUpTimestamp = System.currentTimeMillis();
        }
    }
```
4. isExpired

```java
    /**
     * 检测租约是否过期
     */
    public boolean isExpired(long additionalLeaseMs) {
        //剔除服务时间>0或当前时间>（最后更新时间+租约有效时长+时间偏差）时 表示租约已失效
        return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
    }
```
> 1. renew:针对动作Renew
> 2. cancel:针对动作cancel
> 3. serviceUp:针对动作Register
> 4. isExpired: 用于校验当前实例的租约是否有效，对外提供服务可用性的检测

#### 注册服务基于时间可用性管理

```java
/**
 * 实例租约管理接口，根据租约更新的动作类型操作实例的租约信息
 */
public interface LeaseManager<T> {

    /**
     * 注册实例，更新实例租约信息
     * Assign a new {@link Lease} to the passed in {@link T}.
     */
    void register(T r, int leaseDuration, boolean isReplication);

    /**
     * 剔除实例时，更新实例租约信息
     * Cancel the {@link Lease} associated w/ the passed in <code>appName</code>
     * and <code>id</code>.
     */
    boolean cancel(String appName, String id, boolean isReplication);

    /**
     * 刷新实例时更新实例租约信息
     * Renew the {@link Lease} associated w/ the passed in <code>appName</code>
     * and <code>id</code>.
     */
    boolean renew(String appName, String id, boolean isReplication);

    /**
     * 剔除已过期的租约实例信息
     */
    void evict();
}
```
>LeaseManager接口用于提供服务实例基于时间可控性的管理（租约管理），提供以下功能：
> 1. register ： 服务注册进来设置租约的serviceUp
> 2. cancel : 服务下线，设置实例租约的cancel
> 3. renew ： 服务报活（心跳检测）设置租约的renew
> 4. evict : 遍历注册表通过实例租约isExpired方法校验是否过期，过期强制服务下线操作

> 其中 1-3 是Eureka 客户端发起操作，4,为服务端定时任务轮询服务注册表，主动剔除过期实例。

> LeaseManager默认实现类 AbstractInstanceRegistry

1. register

```java
/**
     * 注册实例，更新实例租约信息
     * Registers a new instance with a given duration.
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            //注册表中查询当前注册服务的集群列表信息
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                //集群不存在时，创建一个新的空集群并添加到注册表
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            //在集群列表中根据服务ID查询实例信息
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                //服务实例信息已存在与集群中
                //根据传入实例与已存在实例的LastDirtyTimestamp判断使用具体实例
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    registrant = existingLease.getHolder();
                }
            } else {
                //服务实例信息不存在与集群中
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfRenewsPerMin > 0) {
                        // Since the client wants to cancel it, reduce the threshold
                        // (1
                        // for 30 seconds, 2 for a minute)
                        this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
                        this.numberOfRenewsPerMinThreshold =
                                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            //根据服务实例信息构建新的租约实例信息
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            //添加服务实例到集群列表
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                //新注册队列中添加当前注册的信息
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            //判断重载状态
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    //将当前服务实例的重载状态添加到重载状态MAP集合中
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            //获取当前实例的重载状态
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                //设置服务的重载状态
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            //基于状态规则重新设置状态
            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // 判断服务注册状态，当状态为UP 更新租约信息
            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            //设置服务实例的动作类型为ADD
            registrant.setActionType(ActionType.ADDED);
            //将服务添加到最近的改变队列
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            //设置服务最后更新时间
            registrant.setLastUpdatedTimestamp();
            //校验缓存信息
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```
 
2. cancel

```java
 /**
     * 根据集群ID ，服务实例ID剔除集群中服务信息
     */
    protected boolean internalCancel(String appName, String id, boolean isReplication) {
        try {
            read.lock();
            CANCEL.increment(isReplication);
            //注册表中获取集群信息列表
            Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
            Lease<InstanceInfo> leaseToCancel = null;
            if (gMap != null) {
                //在就集群列表中根据服务ID移除服务
                leaseToCancel = gMap.remove(id);
            }
            synchronized (recentCanceledQueue) {
                //添加最近下线队列，将当前的下线信息
                recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
            }
            //重载实例状态map集合中移除当前的实例的重载状态
            InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
            if (instanceStatus != null) {
                logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
            }

            if (leaseToCancel == null) {
                CANCEL_NOT_FOUND.increment(isReplication);
                logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
                return false;
            } else {
                //调用租约实例的下线方法，变更租约信息
                leaseToCancel.cancel();
                //获取实例信息
                InstanceInfo instanceInfo = leaseToCancel.getHolder();
                String vip = null;
                String svip = null;
                if (instanceInfo != null) {
                    //设置实例操作类型为DELETED
                    instanceInfo.setActionType(ActionType.DELETED);
                    //最近变更队列中添加当前实例信息
                    recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                    //更新实例的最后更新时间
                    instanceInfo.setLastUpdatedTimestamp();
                    vip = instanceInfo.getVIPAddress();
                    svip = instanceInfo.getSecureVipAddress();
                }
                //校验缓存信息
                invalidateCache(appName, vip, svip);
                logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
                return true;
            }
        } finally {
            read.unlock();
        }
    }
```
3. renew

```java
/**
     * 刷新实例时更新实例租约信息
     */
    public boolean renew(String appName, String id, boolean isReplication) {
        RENEW.increment(isReplication);
        //获取集群列表
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToRenew = null;
        if (gMap != null) {
            //集群列表中查询实例信息
            leaseToRenew = gMap.get(id);
        }
        if (leaseToRenew == null) {
            RENEW_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
            return false;
        } else {
            //获取实例信息
            InstanceInfo instanceInfo = leaseToRenew.getHolder();
            if (instanceInfo != null) {
                // touchASGCache(instanceInfo.getASGName());
                //基于状态规则重新设置状态
                InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                        instanceInfo, leaseToRenew, isReplication);
                if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                    RENEW_NOT_FOUND.increment(isReplication);
                    return false;
                }
                if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                    Object[] args = {
                            instanceInfo.getStatus().name(),
                            instanceInfo.getOverriddenStatus().name(),
                            instanceInfo.getId()
                    };
                    //设置实例状态
                    instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);
                }
            }
            renewsLastMin.increment();
            //租约刷新
            leaseToRenew.renew();
            return true;
        }
    }
```
4. evict

```java
 /**
     * 剔除已过期的租约实例信息
     * @param additionalLeaseMs
     */
    public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");

        if (!isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }

        // We collect first all expired items, to evict them in random order. For large eviction sets,
        // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
        // the impact should be evenly distributed across all applications.
        List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
        //在注册表中查询租约过期的实例列表
        for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
            Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
            if (leaseMap != null) {
                for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                    Lease<InstanceInfo> lease = leaseEntry.getValue();
                    //lease.isExpired() 判断租约是否过期
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }

        // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
        // triggering self-preservation. Without that we would wipe out full registry.
        //获取本地注册表的大小
        int registrySize = (int) getLocalRegistrySize();
        //注册表阈值，由registrySize * 配置百分比参数RenewalPercentThreshold
        int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
        //剔除租约过期的数量 由于RenewalPercentThreshold取值在0-1之间，当为1时，evictionLimit为0
        // 即toEvict为0，每次时不会进行对过期的租约服务下线操作，同时 evictionLimit < expiredLeases.size() 时，
        // 每次操作不会将全部的过期租约实例下线
        int evictionLimit = registrySize - registrySizeThreshold;
        //在租约过期列表大小与每次剔除配置大小中取小值作为本地要剔除的服务实例数
        int toEvict = Math.min(expiredLeases.size(), evictionLimit);
        if (toEvict > 0) {
            logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);
            Random random = new Random(System.currentTimeMillis());
            //遍历剔除服务
            for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
                int next = i + random.nextInt(expiredLeases.size() - i);
                Collections.swap(expiredLeases, i, next);
                Lease<InstanceInfo> lease = expiredLeases.get(i);

                String appName = lease.getHolder().getAppName();
                String id = lease.getHolder().getId();
                EXPIRED.increment();
                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                //调用下线服务方法
                internalCancel(appName, id, false);
            }
        }
    }
```
> AbstractInstanceRegistry类中保存这服务注册表信息，1-3方法为客户端主动发起请求到server端的resources模块，resource接受到客户端的请求，根据请求类型调用
AbstractInstanceRegistry中的1-3方法。

> 4 方法为服务端定时清除注册表中的租约过期实例；在AbstractInstanceRegistry类中如下实现：


```java
  protected void postInit() {
        renewsLastMin.start();
        if (evictionTaskRef.get() != null) {
            evictionTaskRef.get().cancel();
        }
        //创建清除过期租约任务
        evictionTaskRef.set(new EvictionTask());
        //启动定时清除注册表中过期租约信息定时任务
        evictionTimer.schedule(evictionTaskRef.get(),
                serverConfig.getEvictionIntervalTimerInMs(),
                serverConfig.getEvictionIntervalTimerInMs());
    }
```
> postInit 初始化定时任务信息，执行定时清除租约过期任务


```java
class EvictionTask extends TimerTask {

        private final AtomicLong lastExecutionNanosRef = new AtomicLong(0l);

        @Override
        public void run() {
            try {
                long compensationTimeMs = getCompensationTimeMs();
             
                // 执行租约管理方法evict
                evict(compensationTimeMs);
            } catch (Throwable e) {
                logger.error("Could not run the evict task", e);
            }
        }

```
> EvictionTask任务run方法中，执行租约管理的evict方法检测注册表中的过期租约实例，并对其进行下线操作。

> 至此，Eureka赋予了服务注册实例InstanceInfo的基于时间的可用性，并对实例提供了基于时间的可用性的管理。