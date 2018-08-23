title: Eureka之server端集群节点发现，数据同步
author: Mr.G
tags:
  - 微服务
  - Eureka
categories:
  - 微服务
date: 2018-08-23 17:36:00
---
> Eureka服务端支持集群部署，通过源码查看集群节点发现以及数据同步功能的实现

#### 提供集群功能的包路径
> com.netflix.eureka.cluster
#### 集群节点发现以及动态更新节点功能
> Eureka服务端封装了一个集群节点管理的类名称为PeerEurekaNodes 通过名称翻译出来为对等的Eureka节点集合，可以看出这个类是对eureka服务端集群节点抽象，下面通过源码查询eureka是怎么管理与发现节点信息

``` java
/**
 * eureka server 集群节点 集合类 帮助管理维护集群节点
 * Helper class to manage lifecycle of a collection of {@link PeerEurekaNode}s.
 *
 * @author Tomasz Bak
 */
@Singleton
public class PeerEurekaNodes {
    /**
     * 集群节点集合
     */
    private volatile List<PeerEurekaNode> peerEurekaNodes = Collections.emptyList();
    /**
     * 集群节点URL集合
     */
    private volatile Set<String> peerEurekaNodeUrls = Collections.emptySet();

    /**
     * 定时任务线程池
     */
    private ScheduledExecutorService taskExecutor;

```
> 通过PeerEurekaNodes类属性可以看到提供了两个集合以及一个执行定时任务的线程池，其它配置属性忽略
> 1. peerEurekaNodes 表示集群节点集合
> 2. peerEurekaNodeUrls 表示集群节点对应的URL集合
> 3. taskExecutor 执行定时任务的线程池
<!-- more -->
>同时 PeerEurekaNodes 类提供start 与shutdown方法，接下来主要看start方法的实现

``` java
    /**
     * 启动方法，此方法管理集群节点间的通讯
     */
    public void start() {
        //初始化定时任务线程池
        taskExecutor = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                        thread.setDaemon(true);
                        return thread;
                    }
                }
        );
        try {
            updatePeerEurekaNodes(resolvePeerUrls());
            //节点更新任务线程
            Runnable peersUpdateTask = new Runnable() {
                @Override
                public void run() {
                    try {
                        updatePeerEurekaNodes(resolvePeerUrls());
                    } catch (Throwable e) {
                        logger.error("Cannot update the replica Nodes", e);
                    }

                }
            };

//            创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。
//            如果任务的任一执行遇到异常，就会取消后续执行。否则，只能通过执行程序的取消或终止方法来终止该任务。
//            参数：
//            command - 要执行的任务
//            initialdelay - 首次执行的延迟时间
//            delay - 一次执行终止和下一次执行开始之间的延迟
//            unit - initialdelay 和 delay 参数的时间单位
            //定时执行节点更新任务线程
            taskExecutor.scheduleWithFixedDelay(
                    peersUpdateTask,
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                    TimeUnit.MILLISECONDS
            );
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
        for (PeerEurekaNode node : peerEurekaNodes) {
            logger.info("Replica node URL:  " + node.getServiceUrl());
        }
    }
```
> start 方法主要完成以下几件事
> 1. 初始化定时任务线程池
> 2. 首次更新集群节点 updatePeerEurekaNodes方法
> 3. 创建更新集群节点任务线程
> 4. 通过定时任务线程池定时执行更新集群节点线程

> 通过start 可以看出 eureka是通过一个定时线程定时去更新集群的节点信息达到对集群节点的动态发现和感知，在上面我们可以看到更新操作主要由updatePeerEurekaNodes方法完成，下面查看此方法的实现

``` java
    protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
        if (newPeerUrls.isEmpty()) {
            logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
            return;
        }

        Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
        toShutdown.removeAll(newPeerUrls);
        Set<String> toAdd = new HashSet<>(newPeerUrls);
        toAdd.removeAll(peerEurekaNodeUrls);

        //校验新的URL集合与旧有的URL集合是否一致，一致，不需要更新直接返回
        if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
            return;
        }

        // Remove peers no long available
        List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);

        //移除旧集合中不可用的节点信息
        if (!toShutdown.isEmpty()) {
            logger.info("Removing no longer available peer nodes {}", toShutdown);
            int i = 0;
            while (i < newNodeList.size()) {
                PeerEurekaNode eurekaNode = newNodeList.get(i);
                if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                    newNodeList.remove(i);
                    eurekaNode.shutDown();
                } else {
                    i++;
                }
            }
        }

        //添加新增加的节点信息
        // Add new peers
        if (!toAdd.isEmpty()) {
            logger.info("Adding new peer nodes {}", toAdd);
            for (String peerUrl : toAdd) {
                newNodeList.add(createPeerEurekaNode(peerUrl));
            }
        }

        //重新赋值peerEurekaNodes与peerEurekaNodeUrls
        this.peerEurekaNodes = newNodeList;
        this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
    }
```
> updatePeerEurekaNodes根据传入的新集群URL集合完成节点的更新
> 1. 校验传入的URL集合是否需要更新
> 2. 移除新url集合中没有的旧节点并关闭节点
> 3. 创建旧节点集合中没有的新URL节点通过createPeerEurekaNode方法
> 4. 重新赋值节点集合以及URL集合完成节点的更新

>updatePeerEurekaNodes传入的新URL集合是通过resolvePeerUrls方法获取，这个方法实际上是解析配置文件中的eureka.serviceUrl前缀的配置获取，并动态监听配置的更新。 创建新的节点是通过createPeerEurekaNode创建，下面查看此方法源码

``` java
    /**
     * 根据URL创建server新节点信息
     * @param peerEurekaNodeUrl
     * @return
     */
    protected PeerEurekaNode createPeerEurekaNode(String peerEurekaNodeUrl) {
        //创建一个连接远程节点的客户端
        HttpReplicationClient replicationClient = JerseyReplicationClient.createReplicationClient(serverConfig, serverCodecs, peerEurekaNodeUrl);
        //获取新节点host信息
        String targetHost = hostFromUrl(peerEurekaNodeUrl);
        if (targetHost == null) {
            targetHost = "host";
        }
        //创建新节点
        return new PeerEurekaNode(registry, targetHost, peerEurekaNodeUrl, replicationClient, serverConfig);
    }
```
> PeerEurekaNode 方法 
> 1. 创建远程通讯客户端replicationClient 用户与此节点间通讯，数据同步等工作
> 2. 获取要创建的远程节点的host
> 3. 创建一个表示远程节点实例 PeerEurekaNode

> PeerEurekaNode 表示一个与当前节点对等的远程节点，当前节点与远程节点的数据同步工作都是在此实例中完成的。

#### 集群节点数据同步
> 在上面节点发现中知道eureka是通过PeerEurekaNode表示远程对等接点，并将远程通讯客户端replicationClient传入到PeerEurekaNode中，接下来通过查看PeerEurekaNode源码来看eureka集群节点间都有那些数据需要同步以及通讯内容

``` java
    /* For testing */ PeerEurekaNode(PeerAwareInstanceRegistry registry, String targetHost, String serviceUrl,
                                     HttpReplicationClient replicationClient, EurekaServerConfig config,
                                     int batchSize, long maxBatchingDelayMs,
                                     long retrySleepTimeMs, long serverUnavailableSleepTimeMs) {
        String batcherName = getBatcherName();
        //任务处理器
        ReplicationTaskProcessor taskProcessor = new ReplicationTaskProcessor(targetHost, replicationClient);
        //创建一个批量执行的任务调度器
        this.batchingDispatcher = TaskDispatchers.createBatchingTaskDispatcher(
                batcherName,
                config.getMaxElementsInPeerReplicationPool(),
                batchSize,
                config.getMaxThreadsForPeerReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
        //创建一个单任务调度器
        this.nonBatchingDispatcher = TaskDispatchers.createNonBatchingTaskDispatcher(
                targetHost,
                config.getMaxElementsInStatusReplicationPool(),
                config.getMaxThreadsForStatusReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
    }
```
> PeerEurekaNode 完成以下事件
> 1. 创建数据同步的任务处理器ReplicationTaskProcessor
> 2. 创建批处理任务调度器
> 3. 创建单任务处理调度器

> **说明:**  eureka将节点间的数据同步工作包装成一个个细微的任务ReplicationTask ，每一个数据操作代表一个任务，将任务发送给任务调度器TaskDispatcher去异步处理。

> 下来查看PeerEurekaNode都可以创建那些同步任务
- register
 
``` java
    /**
     * 当eureka server注册新服务时，同时创建一个定时任务将新服务同步到集群其它节点
     * Sends the registration information of {@link InstanceInfo} receiving by
     * this node to the peer node represented by this class.
     *
     * @param info
     *            the instance information {@link InstanceInfo} of any instance
     *            that is send to this instance.
     * @throws Exception
     */
    public void register(final InstanceInfo info) throws Exception {
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        //任务调度器中添加一个请求类型为注册register新服务的同步任务
        batchingDispatcher.process(
                taskId("register", info),
                new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.register(info);
                    }
                },
                expiryTime
        );
    }
```
> 注册同步任务，当有服务注册到当前节点时，通过注册同步任务将服务信息同步到集群远程节点

- cancel

``` java
   public void cancel(final String appName, final String id) throws Exception {
        long expiryTime = System.currentTimeMillis() + maxProcessingDelayMs;
        //任务调度器中添加一个请求类型为取消cancel服务的同步任务
        batchingDispatcher.process(
                taskId("cancel", appName, id),
                new InstanceReplicationTask(targetHost, Action.Cancel, appName, id) {
                    @Override
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.cancel(appName, id);
                    }

                    @Override
                    public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
                        super.handleFailure(statusCode, responseEntity);
                        if (statusCode == 404) {
                            logger.warn("{}: missing entry.", getTaskName());
                        }
                    }
                },
                expiryTime
        );
    }
```
> 取消服务注册任务，当前节点有服务取消注册，将信息同步到集群远程节点
- heartbeat

```
    public void heartbeat(final String appName, final String id,
                          final InstanceInfo info, final InstanceStatus overriddenStatus,
                          boolean primeConnection) throws Throwable {
        //当第一次连接时直接发送心跳到远端
        if (primeConnection) {
            // We do not care about the result for priming request.
            replicationClient.sendHeartBeat(appName, id, info, overriddenStatus);
            return;
        }
        //心跳同步任务
        ReplicationTask replicationTask = new InstanceReplicationTask(targetHost, Action.Heartbeat, info, overriddenStatus, false) {
            @Override
            public EurekaHttpResponse<InstanceInfo> execute() throws Throwable {
                return replicationClient.sendHeartBeat(appName, id, info, overriddenStatus);
            }

            @Override
            public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
                super.handleFailure(statusCode, responseEntity);
                if (statusCode == 404) {
                    logger.warn("{}: missing entry.", getTaskName());
                    if (info != null) {
                        logger.warn("{}: cannot find instance id {} and hence replicating the instance with status {}",
                                getTaskName(), info.getId(), info.getStatus());
                        register(info);
                    }
                } else if (config.shouldSyncWhenTimestampDiffers()) {
                    InstanceInfo peerInstanceInfo = (InstanceInfo) responseEntity;
                    if (peerInstanceInfo != null) {
                        syncInstancesIfTimestampDiffers(appName, id, info, peerInstanceInfo);
                    }
                }
            }
        };
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        //任务调度器中添加一个请求类型为heartbeat服务的同步任务
        batchingDispatcher.process(taskId("heartbeat", info), replicationTask, expiryTime);
    }
```
> 心跳同步任务，当前节点有服务发送心跳续租，将信息同步到集群远程节点
- StatusUpdate 
- DeleteStatusOverride

#### 集群节点数据同步任务处理
> 在PeerEurekaNode的构造函数中可以看到同步任务处理由ReplicationTaskProcessor完成，下面看此类源码


``` java
 /**
     * 单个处理ReplicationTask任务
     * @param task
     * @return
     */
    @Override
    public ProcessingResult process(ReplicationTask task) {
        try {
            //调用任务execute方法，完成任务的执行
            EurekaHttpResponse<?> httpResponse = task.execute();
            int statusCode = httpResponse.getStatusCode();
            //判断任务返回结果
            Object entity = httpResponse.getEntity();
            if (logger.isDebugEnabled()) {
                logger.debug("Replication task {} completed with status {}, (includes entity {})", task.getTaskName(), statusCode, entity != null);
            }
            if (isSuccess(statusCode)) {
                task.handleSuccess();
            } else if (statusCode == 503) {
                logger.debug("Server busy (503) reply for task {}", task.getTaskName());
                return ProcessingResult.Congestion;
            } else {
                task.handleFailure(statusCode, entity);
                return ProcessingResult.PermanentError;
            }
        } catch (Throwable e) {
            if (isNetworkConnectException(e)) {
                logNetworkErrorSample(task, e);
                return ProcessingResult.TransientError;
            } else {
                logger.error(peerId + ": " + task.getTaskName() + "Not re-trying this exception because it does not seem to be a network exception", e);
                return ProcessingResult.PermanentError;
            }
        }
        return ProcessingResult.Success;
    }
```
> 单任务处理 
> 1. 调用任务task的execute完成远程数据同步
> 2. 分析远程返回结果


``` java
 /**
     * 批量处理ReplicationTask任务
     * @param tasks
     * @return
     */
    @Override
    public ProcessingResult process(List<ReplicationTask> tasks) {
        //根据task集合创建ReplicationList
        ReplicationList list = createReplicationListOf(tasks);
        try {
            //调用批量同步接口 将同步集合发送到远端节点同步数据
            EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
            //判断同步返回结果
            int statusCode = response.getStatusCode();
            if (!isSuccess(statusCode)) {
                if (statusCode == 503) {
                    logger.warn("Server busy (503) HTTP status code received from the peer {}; rescheduling tasks after delay", peerId);
                    return ProcessingResult.Congestion;
                } else {
                    // Unexpected error returned from the server. This should ideally never happen.
                    logger.error("Batch update failure with HTTP status code {}; discarding {} replication tasks", statusCode, tasks.size());
                    return ProcessingResult.PermanentError;
                }
            } else {
                handleBatchResponse(tasks, response.getEntity().getResponseList());
            }
        } catch (Throwable e) {
            if (isNetworkConnectException(e)) {
                logNetworkErrorSample(null, e);
                return ProcessingResult.TransientError;
            } else {
                logger.error("Not re-trying this exception because it does not seem to be a network exception", e);
                return ProcessingResult.PermanentError;
            }
        }
        return ProcessingResult.Success;
    }
```
> 批处理任务，将一组任务一次性发送到远程进行处理
> 1. 根据task集合创建ReplicationList
> 2. 调用批量同步接口将同步集合发送到远端节点同步数据 即调用rest API /{version}/peerreplication
> 3. 分析远程返回结果

---

> **eureka 服务端 集群节点发现，数据同步功能主要是由PeerEurekaNodes与PeerEurekaNode类实现,通过源码的跟踪可以清晰看出集群实现的逻辑，方便在实际应用中对问题的定位**