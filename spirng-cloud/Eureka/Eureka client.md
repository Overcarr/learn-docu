#                                        0.Eureka client

# 1.核心类

## 1.1.EurekaClientAutoConfigureatione

 Eureka client的核心启动类。

###  1.1.1.注入EurekaClient

 A、B只会有一个被注入到Spring容器中，若是B存在刷新功能，是多例的、且是懒加载，A是单例的 ---- 注解的原因

```java
A、
@Configuration(
        proxyBeanMethods = false
    )
    @EurekaClientAutoConfiguration.ConditionalOnMissingRefreshScope
    protected static class EurekaClientConfiguration {
        @Autowired
        private ApplicationContext context;
        @Autowired
        private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

        protected EurekaClientConfiguration() {
        }

        @Bean(
            destroyMethod = "shutdown"
        )
        @ConditionalOnMissingBean(
            value = {EurekaClient.class},
            search = SearchStrategy.CURRENT
        )
        public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config) {
            return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
        }
      
----
B、
@Configuration(
        proxyBeanMethods = false
    )
    @EurekaClientAutoConfiguration.ConditionalOnRefreshScope
    protected static class RefreshableEurekaClientConfiguration {
        @Autowired
        private ApplicationContext context;
        @Autowired
        private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

        protected RefreshableEurekaClientConfiguration() {
        }

        @Bean(
            destroyMethod = "shutdown"
        )
        @ConditionalOnMissingBean(
            value = {EurekaClient.class},
            search = SearchStrategy.CURRENT
        )
        @org.springframework.cloud.context.config.annotation.RefreshScope
        @Lazy
        public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config, EurekaInstanceConfig instance, @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
            ApplicationInfoManager appManager;
            if (AopUtils.isAopProxy(manager)) {
                appManager = (ApplicationInfoManager)ProxyUtils.getTargetObject(manager);
            } else {
                appManager = manager;
            }

            CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager, config, this.optionalArgs, this.context);
            cloudEurekaClient.registerHealthCheck(healthCheckHandler);
            return cloudEurekaClient;
        }
```

### 1.1.2.

## 1.2.InstanceInfo

封装主机实例信息

​        1、两个时间戳

```
1、lastUpdatedTimestamp
   记录 instanceInfo 在Server 端被修改的时间。这个字段只会在server端被更新，server端通常只会修改 instanceInfo 的状态信息（我暂时只发现只修改了状态信息）

2、lastDirtyTimestamp
    记录 instanceInfo 在 Client 端被修改的时间。这个字段只会在client端被更新， instanceInfo 中任何信息被修改（数据中心信息、续约配置信息、状态等），都会更新该时间戳。
```

​       2、两种状态

```
1、status
    InstanceInfo代表一个主机实例信息，如果客户端向服务端注册了，那么服务端和客户端都会维护一个InstanceInfo信息。 
客户端的InstanceInfo代表了自己，服务端的InstanceInfo是在注册表中该客户端的注册信息。 
客户端的InstanceInfo的status代表的是这个客户端真正的工作状态，而服务端的InstanceInfo的status，是在进行服务发现时希望暴露给其他客户端的工作状态。

所以同一个客户端的InstanceInfo，在客户端本地的status和在服务端注册表中的status并不一定是完全一样的。 
那么客户端如何去修改服务端注册表中它的实例信息的status？就是通过overriddenStatus，客户端发起修改状态请求传递给服务的状态就是通过overriddenStatus字段，

服务端收到overriddenStatus会将它保存起来，并会根据一套覆盖状态的规则，计算出真正的InstanceInfo在服务端注册表中的status， 
服务端注册表中的status并不会影响到客户端自己的status，并且客户端之后发起的所有向服务端同步数据的请求，服务端在进行状态处理时，并不会直接拿客户端传过来的status更新，而是根据覆盖状态的规则计算出status。 

2、overriddenStatus
```

## 1.3.Application

所有同一个微服务名称的主机实例都会维护在这个类

```java
public class Application {
    
    private static Random shuffleRandom = new Random();

    @Override
    public String toString() {
        return "Application [name=" + name + ", isDirty=" + isDirty
                + ", instances=" + instances + ", shuffledInstances="
                + shuffledInstances + ", instancesMap=" + instancesMap + "]";
    }

    private String name;

    @XStreamOmitField
    private volatile boolean isDirty = false;
    // 实例集合
    @XStreamImplicit
    private final Set<InstanceInfo> instances;

    private final AtomicReference<List<InstanceInfo>> shuffledInstances;
    // 实例集合 key id value 具体实例
    private final Map<String, InstanceInfo> instancesMap;
```

## 1.4.ApplicationsApplications

注册表，维护的是整个从服务端获取的注册表信息

```java
public class Applications {
    private static class VipIndexSupport {
        final AbstractQueue<InstanceInfo> instances = new ConcurrentLinkedQueue<>();
        final AtomicLong roundRobinIndex = new AtomicLong(0);
        final AtomicReference<List<InstanceInfo>> vipList = new AtomicReference<>(Collections.emptyList());

        public AtomicLong getRoundRobinIndex() {
            return roundRobinIndex;
        }

        public AtomicReference<List<InstanceInfo>> getVipList() {
            return vipList;
        }
    }

    private static final String STATUS_DELIMITER = "_";

    private String appsHashCode;
    private Long versionDelta;
    @XStreamImplicit
    private final AbstractQueue<Application> applications;
    private final Map<String, Application> appNameApplicationMap;
    private final Map<String, VipIndexSupport> virtualHostNameAppMap;
    private final Map<String, VipIndexSupport> secureVirtualHostNameAppMap;
```

## 1.4.Jersey 框架

Client与Server之间的通信，处理器是Resource



# 2.初始化执行构造器

## 2.1.CloudEurekaClient

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args, ApplicationEventPublisher publisher) {
        // 调用父类加载器
        super(applicationInfoManager, config, args);
        this.cacheRefreshedCount = new AtomicLong(0L);
        this.eurekaHttpClient = new AtomicReference();
        this.applicationInfoManager = applicationInfoManager;
        this.publisher = publisher;
        this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
        ReflectionUtils.makeAccessible(this.eurekaTransportField);
    }
```

## 2.2.DiscoveryClient：构造方法

```java
// A、获取注册表
if (clientConfig.shouldFetchRegistry()) {
            try {
                boolean primaryFetchRegistryResult = fetchRegistry(false);
                if (!primaryFetchRegistryResult) {
                    logger.info("Initial registry fetch from primary servers failed");
                }
                boolean backupFetchRegistryResult = true;
                if (!primaryFetchRegistryResult && !fetchRegistryFromBackup()) {
                    backupFetchRegistryResult = false;
                    logger.info("Initial registry fetch from backup servers failed");
                }
                if (!primaryFetchRegistryResult && !backupFetchRegistryResult && clientConfig.shouldEnforceFetchRegistryAtInit()) {
                    throw new IllegalStateException("Fetch registry error at startup. Initial fetch failed.");
                }
            } catch (Throwable th) {
                logger.error("Fetch registry error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // call and execute the pre registration handler before all background tasks (inc registration) is started
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        }
        // B、注册
        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // C、初始化定时任务
        initScheduledTasks();
```

### 2.2.1.获取注册表

```java
// eureka.client.fetch-registry  配置是否拉取注册表               
if (clientConfig.shouldFetchRegistry()) {
            try {
                boolean primaryFetchRegistryResult = fetchRegistry(false);
                if (!primaryFetchRegistryResult) {
                    logger.info("Initial registry fetch from primary servers failed");
                }
                boolean backupFetchRegistryResult = true;
                if (!primaryFetchRegistryResult && !fetchRegistryFromBackup()) {
                    backupFetchRegistryResult = false;
                    logger.info("Initial registry fetch from backup servers failed");
                }
                if (!primaryFetchRegistryResult && !backupFetchRegistryResult && clientConfig.shouldEnforceFetchRegistryAtInit()) {
                    throw new IllegalStateException("Fetch registry error at startup. Initial fetch failed.");
                }
            } catch (Throwable th) {
                logger.error("Fetch registry error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }   
```

#### 主获取注册表方法

```java
// 入参：ture全量复制，fales可能全量、可能增量 
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {                       (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                // 满足一个条件即 全量复制
                getAndStoreFullRegistry();
            } else {
                // 增量复制
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
                    appPathIdentifier, clientConfig.getRegistryFetchIntervalSeconds(), e.getMessage(), ExceptionUtils.getStackTrace(e));
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }
```

全量复制

```java
private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        // 根据是否是VIP地址，选择一个，调用Jsrsey提交获取注册表请求 
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
          // 将获取到的注册表缓存到本地 
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
```

增量复制

#### 备获取注册表方法

```java
private boolean fetchRegistryFromBackup() {
        try {
            @SuppressWarnings("deprecation")
            BackupRegistry backupRegistryInstance = newBackupRegistryInstance();
            if (null == backupRegistryInstance) { // backward compatibility with the old protected method, in case it is being used.
                backupRegistryInstance = backupRegistryProvider.get();
            }
            
            if (null != backupRegistryInstance) {
                Applications apps = null;
              // 从远程Region获取注册表
                if (isFetchingRemoteRegionRegistries()) {
                    String remoteRegionsStr = remoteRegionsToFetch.get();
                    if (null != remoteRegionsStr) {
                        apps = backupRegistryInstance.fetchRegistry(remoteRegionsStr.split(","));
                    }
                } else {
                  // 从备用注册表提供者获取注册表
                    apps = backupRegistryInstance.fetchRegistry();
                }
                if (apps != null) {
                  // 注册表缓存到本地
                    final Applications applications = this.filterAndShuffle(apps);
                    applications.setAppsHashCode(applications.getReconcileHashCode());
                    localRegionApps.set(applications);
                    logTotalInstances();
                    logger.info("Fetched registry successfully from the backup");
                    return true;
                }
            } else {
                logger.warn("No backup registry instance defined & unable to find any discovery servers.");
            }
        } catch (Throwable e) {
            logger.warn("Cannot fetch applications from apps although backup registry was specified", e);
        }
        return false;
    }
```

### 2.2.2.注册

```java
// 1、Eureka.client. register-with-eureka:   是否注册到Eureka 
// 2、 Eureka.client. should-enforce-registration-at-init:  是否初始化的时候注册到Eureka  默认为fales。所以这里不会注册 
if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }
```

```java
// 调用Jsrsey提交注册POST请求
@Override
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
```

从代码中可以看到，如果设置为初始化时注册，会一个问题：

如果注册失败了会抛异常，导致整个客户端启动就失败了，也不会启动后面的定时任务，如果不强制初始化时进行注册，会通过心跳续约的定时任务去注册，即使注册失败了也不影响客户端启动，并会定时多次尝试进行注册。

### 2.2.3.初始化定时任务

initScheduledTasks().

#### 初始化拉取注册表定时任务

```java
private void initScheduledTasks() {
        // Eureka.client. fetch-registry: 是否拉取注册表 
        if (clientConfig.shouldFetchRegistry()) {
            // Euerka.client.registry-fetch-interval-seconds:拉取注册表间隔 
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
          // Eureka.client.cache-refresh-executor-exponential-back-off-bound: 最大延迟拉取注册表的倍数 
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            cacheRefreshTask = new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new CacheRefreshThread()
            );
          // 一次性的定时任务 
            scheduler.schedule(
                    cacheRefreshTask,
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
  
  // 
```

调用scheduler.schedule --> 执行TimedSupervisoTask任务

```java
TimedSupervisorTask.java  
@Override
    public void run() {
        Future<?> future = null;
        try {
          // 异步执行拉取注册表
            future = executor.submit(task);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
          // 获取异步执行结果，阻塞
            future.get(timeoutMillis, TimeUnit.MILLISECONDS);  // block until done or timeout
          // 异步获取成功，设置下一次获取注册表的执行间隔为初始值
            delay.set(timeoutMillis);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            successCounter.increment();
        } catch (TimeoutException e) {
          // 异步执行超时或失败
            logger.warn("task supervisor timed out", e);
            timeoutCounter.increment();
          // 设置下一次获取注册表的执行间隔
            long currentDelay = delay.get();
            long newDelay = Math.min(maxDelay, currentDelay * 2);
            delay.compareAndSet(currentDelay, newDelay);

        } catch (RejectedExecutionException e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, reject the task", e);
            } else {
                logger.warn("task supervisor rejected the task", e);
            }

            rejectedCounter.increment();
        } catch (Throwable e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, can't accept the task");
            } else {
                logger.warn("task supervisor threw an exception", e);
            }

            throwableCounter.increment();
        } finally {
            if (future != null) {
                future.cancel(true);
            }
            // 再次调用定时任务
            if (!scheduler.isShutdown()) {
                scheduler.schedule(this, delay.get(), TimeUnit.MILLISECONDS);
            }
        }
    }
```

future = executor.submit(task) --> 执行CacheRefreshThread.run

```java
class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }

    @VisibleForTesting
    void refreshRegistry() {
        try {
            boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

            boolean remoteRegionsModified = false;
            // 判断远程Region是否已被修改 如果已被修改 全量复制
            String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
            if (null != latestRemoteRegions) {
                String currentRemoteRegions = remoteRegionsToFetch.get();
                if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                    // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                    synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                        if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                            String[] remoteRegions = latestRemoteRegions.split(",");
                            remoteRegionsRef.set(remoteRegions);
                            instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                            remoteRegionsModified = true;
                        } else {
                            logger.info("Remote regions to fetch modified concurrently," +
                                    " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                        }
                    }
                } else {
                    // Just refresh mapping to reflect any DNS/Property change
                    instanceRegionChecker.getAzToRegionMapper().refreshMapping();
                }
            }

          // 拉取注册表
            boolean success = fetchRegistry(remoteRegionsModified);
            if (success) {
                registrySize = localRegionApps.get().size();
                lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
            }

            if (logger.isDebugEnabled()) {
                StringBuilder allAppsHashCodes = new StringBuilder();
                allAppsHashCodes.append("Local region apps hashcode: ");
                allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
                allAppsHashCodes.append(", is fetching remote regions? ");
                allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
                for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                    allAppsHashCodes.append(", Remote region: ");
                    allAppsHashCodes.append(entry.getKey());
                    allAppsHashCodes.append(" , apps hashcode: ");
                    allAppsHashCodes.append(entry.getValue().getAppsHashCode());
                }
                logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                        allAppsHashCodes);
            }
        } catch (Throwable e) {
            logger.error("Cannot fetch registry from server", e);
        }
    }
```

fetchRegistry(remoteRegionsModified);.

1.全量复制

如果没有强制全量复制，或者本地缓存注册表不为空，则增量复制

2.增量复制

```java
private void getAndUpdateDelta(Applications applications) throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        Applications delta = null;
  // 远程调用接口，获取增量注册表信息
        EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            delta = httpResponse.getEntity();
        }
        // 如果获取结果为空，则全量复制
        if (delta == null) {
            logger.warn("The server does not allow the delta revision to be applied because it is not safe. "
                    + "Hence got the full registry.");
            getAndStoreFullRegistry();
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {         // 如果结果不为空，则CAS把增量注册表信息缓存到本地
            logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
            String reconcileHashCode = "";
            if (fetchRegistryUpdateLock.tryLock()) {
                try {
                    updateDelta(delta);
                    reconcileHashCode = getReconcileHashCode(applications);
                } finally {
                    fetchRegistryUpdateLock.unlock();
                }
            } else {
                logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
            }
            // 判断delta的hashCode和reconcileHashCode是否相等，如果不相等，则走全量复制
            if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
                reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
            }
        } else {
            logger.warn("Not updating application delta as another thread is updating it already");
            logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
        }
    }

// 增量更新到本地注册表
private void updateDelta(Applications delta) {
        int deltaCount = 0;
        for (Application app : delta.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                Applications applications = getApplications();
                String instanceRegion = instanceRegionChecker.getInstanceRegion(instance);
              // 判断当前Region是否在本地存在，如果本地不存在这个Region，则创建一个新的。
                if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                    Applications remoteApps = remoteRegionVsApps.get(instanceRegion);
                    if (null == remoteApps) {
                        remoteApps = new Applications();
                        remoteRegionVsApps.put(instanceRegion, remoteApps);
                    }
                    applications = remoteApps;
                }

                ++deltaCount;
              // 新增主机实例(instance)，并且标记为脏数据
                if (ActionType.ADDED.equals(instance.getActionType())) {
                    Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                    if (existingApp == null) {
                        applications.addApplication(app);
                    }
                    logger.debug("Added instance {} to the existing apps in region {}", instance.getId(), instanceRegion);
                    applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);
                } else if (ActionType.MODIFIED.equals(instance.getActionType())) {
                  // 修改主机实例(instance)，并且标记为脏数据
                    Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                    if (existingApp == null) {
                        applications.addApplication(app);
                    }
                    logger.debug("Modified instance {} to the existing apps ", instance.getId());

                    applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);

                } else if (ActionType.DELETED.equals(instance.getActionType())) {
                  // 删除主机实例(instance)
                    Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                    if (existingApp != null) {
                        logger.debug("Deleted instance {} to the existing apps ", instance.getId());
                        existingApp.removeInstance(instance);
                      // 删除主机实例之后，判断此Application微服务是否为空，若为空则从本地注册表中删除此Application。
                        if (existingApp.getInstancesAsIsFromEureka().isEmpty()) {
                            applications.removeApplication(existingApp);
                        }
                    }
                }
            }
        }
        logger.debug("The total number of instances fetched by the delta processor : {}", deltaCount);

        getApplications().setVersion(delta.getVersion());
        getApplications().shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());

        for (Applications applications : remoteRegionVsApps.values()) {
            applications.setVersion(delta.getVersion());
            applications.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
        }
    }
```



#### 初始化心跳续约定时任务

```java
if (clientConfig.shouldRegisterWithEureka()) {
            // 心跳周期
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            // Eureka.client. heartbeat-executor-exponential-back-off-bound: 心跳超时以后下次延迟最大的倍数 
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            heartbeatTask = new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new HeartbeatThread()
            );
          // 执行一次性任务 
            scheduler.schedule(
                    heartbeatTask,
                    renewalIntervalInSecs, TimeUnit.SECONDS);
```

调用scheduler.schedule --> 执行TimedSupervisorTask.run任务

```java
@Override
    public void run() {
        Future<?> future = null;
        try {
            future = executor.submit(task);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            future.get(timeoutMillis, TimeUnit.MILLISECONDS);  // block until done or timeout
            delay.set(timeoutMillis);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            successCounter.increment();
        } catch (TimeoutException e) {
            logger.warn("task supervisor timed out", e);
            timeoutCounter.increment();

            long currentDelay = delay.get();
            long newDelay = Math.min(maxDelay, currentDelay * 2);
            delay.compareAndSet(currentDelay, newDelay);

        } catch (RejectedExecutionException e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, reject the task", e);
            } else {
                logger.warn("task supervisor rejected the task", e);
            }

            rejectedCounter.increment();
        } catch (Throwable e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, can't accept the task");
            } else {
                logger.warn("task supervisor threw an exception", e);
            }

            throwableCounter.increment();
        } finally {
            if (future != null) {
                future.cancel(true);
            }

            if (!scheduler.isShutdown()) {
                scheduler.schedule(this, delay.get(), TimeUnit.MILLISECONDS);
            }
        }
    }
```

future = executor.submit(task) --> 执行HeartbeatThread.run

```java
private class HeartbeatThread implements Runnable {

        public void run() {
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
    }
    
    
     boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
          // 调用远程接口，执行心跳续期。
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
              // 如果返回结果404则，调用注册方法，注册该实例到Server。
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                  // 如果注册成功，判断是否需要取消脏数据标志
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
          // 返回成功续约心跳结果
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }

// InstanceInfo.java
// 如果脏数据时间大于等于当前实例的最后脏数据修改时间，则把脏数据标志设置为fales。
// 如果小于，情况是 ，在我们设置脏数据时间到注册这段时间内，实例的数据存在更新，则脏数据时间再次被更新了，所以server跟client是存在脏数据的，数据存在差异的，是需要复制同步的。
public synchronized void unsetIsDirty(long unsetDirtyTimestamp) {
        if (lastDirtyTimestamp <= unsetDirtyTimestamp) {
            isInstanceInfoDirty = false;
        } else {
        }
    }
```



#### 初始化Client更新任务

```java
// InstanceInfo replicator
            // 复制同步任务
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            // 开启状态监听器
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
```

1.复制同步任务

```java
// 作用 ：更新、复制同步
 
// 配置了一个线程，说明会通过这个线程定时检测client端的数据更新并将更新的数据同步给服务端
// onDemandUpdate方法，可以按需随时检测并向服务端进行同步，比如上面就看到的状态变更监听器，一但监听到状态变更，就触发了该方法
 
// 后面两句话，说明了两个特性：
// 速率限制，限制按需执行的频率，避免频繁向Server端发起同步，底层实现用的RateLimiter，基于令牌桶算法的速率限制器
// onDemandUpdate方法，按需执行的方式会中断定时任务，在onDemandUpdate方法结束后重新开启定时任务
class InstanceInfoReplicator implements Runnable {
    private static final Logger logger = LoggerFactory.getLogger(InstanceInfoReplicator.class);

    private final DiscoveryClient discoveryClient;
    private final InstanceInfo instanceInfo;

    private final int replicationIntervalSeconds;
    private final ScheduledExecutorService scheduler;
    private final AtomicReference<Future> scheduledPeriodicRef;

    private final AtomicBoolean started;
    private final RateLimiter rateLimiter;
    private final int burstSize;
    private final int allowedRatePerMinute;
  
  InstanceInfoReplicator(DiscoveryClient discoveryClient, InstanceInfo instanceInfo, int replicationIntervalSeconds, int burstSize) {
        this.discoveryClient = discoveryClient;
        this.instanceInfo = instanceInfo;
       // 初始化一个执行器，专门执行 client复制同步任务
        this.scheduler = Executors.newScheduledThreadPool(1,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-InstanceInfoReplicator-%d")
                        .setDaemon(true)
                        .build());

        this.scheduledPeriodicRef = new AtomicReference<Future>();

        this.started = new AtomicBoolean(false);
        this.rateLimiter = new RateLimiter(TimeUnit.MINUTES);
        this.replicationIntervalSeconds = replicationIntervalSeconds;
        this.burstSize = burstSize;
       // 允许每分钟变化率，主要控制，按需执行的方式，过于频繁。
        this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
        logger.info("InstanceInfoReplicator onDemand update allowed rate per min is {}", allowedRatePerMinute);
    }

    public void start(int initialDelayMs) {
      // CAS开启执行器
      // 判断现在starter的状态是否是fales，未开启定时任务。
      // 标志client为脏数据。启动定时任务，注意scheduler.schedule只会执行一次任务，返回异步操作结果。将异步结果保存。
        if (started.compareAndSet(false, true)) {
            instanceInfo.setIsDirty();  // for initial register
            Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
}
```

scheduler.schedule --> 执行InstanceInfoReplicator.run

```java
public void run() {
        try {
          // 获取更新内容逻辑
            discoveryClient.refreshInstanceInfo();

          // 如果存在脏数据时间，则需要重新注册，注册完成以后判断是否能把脏数据标志设置为fales
            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
          // 任务完成以后，重新定时启动任务，达到一直循环执行任务的目的
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
```

获取更新内容逻辑

```java
void refreshInstanceInfo() {
        // 刷新数据中心信息
        applicationInfoManager.refreshDataCenterInfoIfRequired();
       // 刷新心跳续约信息
        applicationInfoManager.refreshLeaseInfoIfRequired();

       // 检查健康状态
        InstanceStatus status;
        try {
            status = getHealthCheckHandler().getStatus(instanceInfo.getStatus());
        } catch (Exception e) {
            logger.warn("Exception from healthcheckHandler.getStatus, setting status to DOWN", e);
            status = InstanceStatus.DOWN;
        }

        if (null != status) {
            applicationInfoManager.setInstanceStatus(status);
        }
    }

```

刷新心跳续约信息

```java
 public void refreshLeaseInfoIfRequired() {
        LeaseInfo leaseInfo = instanceInfo.getLeaseInfo();
        if (leaseInfo == null) {
            return;
        }
   // 从配置中心获取：心跳间隔时间及多长时间收不到心跳时间即认为下线 两个值
        int currentLeaseDuration = config.getLeaseExpirationDurationInSeconds();
        int currentLeaseRenewal = config.getLeaseRenewalIntervalInSeconds();
   // 如果这两个值跟类中的值存在差异
        if (leaseInfo.getDurationInSecs() != currentLeaseDuration || leaseInfo.getRenewalIntervalInSecs() != currentLeaseRenewal) {
          // 则重新赋值，设置为脏数据标志。
            LeaseInfo newLeaseInfo = LeaseInfo.Builder.newBuilder()
                    .setRenewalIntervalInSecs(currentLeaseRenewal)
                    .setDurationInSecs(currentLeaseDuration)
                    .build();
            instanceInfo.setLeaseInfo(newLeaseInfo);
            instanceInfo.setIsDirty();
        }
    }
```

2.状态监听器

开启一个状态监听器，一旦发现实例状态有变化即调用onDemandUpdate()。

```java
// 开启状态监听器
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

// InstanceInfoReplicator.java
  public boolean onDemandUpdate() {
    // 基于令牌桶算法的速率限制器，限制按需执行方法调用太频繁，第一个参数：允许以突发形式进入系统的请求的最大数量，第二个参数：期望的每秒请求数(也支持使用分钟的速率限制器)
        if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
            if (!scheduler.isShutdown()) {
                scheduler.submit(new Runnable() {
                    @Override
                    public void run() {
                      // 任务执行器开启一个线程，获取最近一次异步执行结果，如果任务没有完成，则取消这个任务
                        logger.debug("Executing on-demand update of local InstanceInfo");
    
                        Future latestPeriodic = scheduledPeriodicRef.get();
                        if (latestPeriodic != null && !latestPeriodic.isDone()) {
                            logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                            latestPeriodic.cancel(false);
                        }
                        // 执行run()
                        InstanceInfoReplicator.this.run();
                    }
                });
                return true;
            } else {
                logger.warn("Ignoring onDemand update due to stopped scheduler");
                return false;
            }
        } else {
            logger.warn("Ignoring onDemand update due to rate limiter");
            return false;
        }
    }
```



# 3.下架

当关闭实例应用程序时，即调用Eureka client的shutdown()方法

```java
@Bean(
            destroyMethod = "shutdown"
        )
        @ConditionalOnMissingBean(
            value = {EurekaClient.class},
            search = SearchStrategy.CURRENT
        )
        @org.springframework.cloud.context.config.annotation.RefreshScope
        @Lazy
        public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config, EurekaInstanceConfig instance, @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
            ApplicationInfoManager appManager;
            if (AopUtils.isAopProxy(manager)) {
                appManager = (ApplicationInfoManager)ProxyUtils.getTargetObject(manager);
            } else {
                appManager = manager;
            }

            CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager, config, this.optionalArgs, this.context);
            cloudEurekaClient.registerHealthCheck(healthCheckHandler);
            return cloudEurekaClient;
        }

// Discoveryclient
@PreDestroy
    @Override
    public synchronized void shutdown() {
      // CAS保证线程安全
        if (isShutdown.compareAndSet(false, true)) {
            logger.info("Shutting down DiscoveryClient ...");
            // 注销状态监听器
            if (statusChangeListener != null && applicationInfoManager != null) {
                applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
            }
            // 下架定时任务
            cancelScheduledTasks();

            // If APPINFO was registered
            if (applicationInfoManager != null
                    && clientConfig.shouldRegisterWithEureka()
                    && clientConfig.shouldUnregisterOnShutdown()) {
                applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
              // Server端注销该实例
                unregister();
            }

            if (eurekaTransport != null) {
                eurekaTransport.shutdown();
            }

            heartbeatStalenessMonitor.shutdown();
            registryStalenessMonitor.shutdown();

            Monitors.unregisterObject(this);

            logger.info("Completed shut down of DiscoveryClient");
        }
    }
```



# 修改实例状态

整合Actuator才可

```java

@Endpoint(
    id = "serviceregistry"
)
public class ServiceRegistryEndpoint {
    private final ServiceRegistry serviceRegistry;
    private Registration registration;

    public ServiceRegistryEndpoint(ServiceRegistry<?> serviceRegistry) {
        this.serviceRegistry = serviceRegistry;
    }

    public void setRegistration(Registration registration) {
        this.registration = registration;
    }

    @WriteOperation
    public ResponseEntity<?> setStatus(String status) {
        Assert.notNull(status, "status may not by null");
        if (this.registration == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body("no registration found");
        } else {
          // 当调用请求更改状态，就会触发setStatus方法
            this.serviceRegistry.setStatus(this.registration, status);
            return ResponseEntity.ok().build();
        }
    }

    @ReadOperation
    public ResponseEntity getStatus() {
        return this.registration == null ? ResponseEntity.status(HttpStatus.NOT_FOUND).body("no registration found") : ResponseEntity.ok().body(this.serviceRegistry.getStatus(this.registration));
    }
}

//EurekaServiceRegistry
public void setStatus(EurekaRegistration registration, String status) {
        InstanceInfo info = registration.getApplicationInfoManager().getInfo();
        if ("CANCEL_OVERRIDE".equalsIgnoreCase(status)) {
          // 下架实例
            registration.getEurekaClient().cancelOverrideStatus(info);
        } else {
          // 修改实例状态
            InstanceStatus newStatus = InstanceStatus.toEnum(status);
            registration.getEurekaClient().setStatus(newStatus, info);
        }
    }
```

