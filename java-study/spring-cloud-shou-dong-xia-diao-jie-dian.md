---
description: spring cloud手动下掉节点
---

# spring cloud手动下掉节点

## 背景

项目中使用nacos作为注册中心，在服务发布过程中，经常会遇到服务发布过程中实例重启过程中还会有流量打向实例A，对业务造成一定影响

## 目标

实现服务的平滑重启

## 方案

### 现状介绍

当前服务是部署在虚拟机上的，在服务重启时，使用systemctl命令实现服务重启

```bash
systemctl restart xxxx
```

重启过程中，会先向服务发送SIGTERM(15)信号，然后等待一段时间之后再发送SIGKILL(9)

SIGTERM信号java程序可以通过Runtime类监听，在程序结束之前做一些善后工作

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try {
                doSomeThing();
            } catch (Exception ex) {
                log.error("doSomeThing error. ", ex);
            }

        }));
```

事实上spring已经有这部分工作，参考

org.springframework.boot.SpringApplicationShutdownHook

<pre class="language-java"><code class="lang-java">class SpringApplicationShutdownHook implements Runnable {
	// ...
<strong>	void registerApplicationContext(ConfigurableApplicationContext context) {
</strong>		addRuntimeShutdownHookIfNecessary();
		synchronized (SpringApplicationShutdownHook.class) {
			assertNotInProgress();
			context.addApplicationListener(this.contextCloseListener);
			this.contexts.add(context);
		}
	}

	private void addRuntimeShutdownHookIfNecessary() {
		if (this.shutdownHookAdded.compareAndSet(false, true)) {
			addRuntimeShutdownHook();
		}
	}

	void addRuntimeShutdownHook() {
		try {
			Runtime.getRuntime().addShutdownHook(new Thread(this, "SpringApplicationShutdownHook"));
		}
		catch (AccessControlException ex) {
			// Not allowed in some environments
		}
	}

	@Override
	public void run() {
		Set&#x3C;ConfigurableApplicationContext> contexts;
		Set&#x3C;ConfigurableApplicationContext> closedContexts;
		Set&#x3C;Runnable> actions;
		synchronized (SpringApplicationShutdownHook.class) {
			this.inProgress = true;
			contexts = new LinkedHashSet&#x3C;>(this.contexts);
			closedContexts = new LinkedHashSet&#x3C;>(this.closedContexts);
			actions = new LinkedHashSet&#x3C;>(this.handlers.getActions());
		}
		contexts.forEach(this::closeAndWait);
		closedContexts.forEach(this::closeAndWait);
		actions.forEach(Runnable::run);
	}
	//...
}
</code></pre>

spring cloud注册中心实现

参考org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#destroy

```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
		implements AutoServiceRegistration, ApplicationContextAware, ApplicationListener<WebServerInitializedEvent> {

	// ...
	public void onApplicationEvent(WebServerInitializedEvent event) {
		bind(event);
	}

	@Deprecated
	public void bind(WebServerInitializedEvent event) {
		ApplicationContext context = event.getApplicationContext();
		if (context instanceof ConfigurableWebServerApplicationContext) {
			if ("management".equals(((ConfigurableWebServerApplicationContext) context).getServerNamespace())) {
				return;
			}
		}
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}
	// ...
	public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get()) {
			this.context.publishEvent(new InstancePreRegisteredEvent(this, getRegistration()));
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}

	}
	// ...
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}
	
	// ...
	@PreDestroy
	public void destroy() {
		stop();
	}
	// ...
	public void stop() {
		if (this.getRunning().compareAndSet(true, false) && isEnabled()) {
			deregister();
			if (shouldRegisterManagement()) {
				deregisterManagement();
			}
			this.serviceRegistry.close();
		}
	}

```

通过监听spring WebServerInitializedEvent实现服务的注册，见onApplicationEvent；通过bean销毁钩子实现服务的下线，参考destroy方法，方法上有@PreDestroy注解，会在bean被销毁前调用。

nacos实现了该类，参考com.alibaba.cloud.nacos.registry.NacosServiceRegistry；也就是说nacos会在服务销毁之前将当前实例从nacos上下掉

### 为什么重启过车中还会有流量？

#### 服务销毁前，已经从上有打过来的未处理完的流量

当前spring boot 已经配置了平滑销毁，具体配置如下

```yaml
server:
  port: 8801
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

理论上销毁前会将未处理完的请求处理完之后再销毁服务，所以这种可能排除

#### 服务从nacos上下掉之后，上游本地的缓存还未更新，导致上游未及时感知到下游提供方的变更

nacos discover client实现参考com.alibaba.cloud.nacos.discovery.NacosDiscoveryClient

```java
public class NacosDiscoveryClient implements DiscoveryClient {
	// ...
	@Override
	public List<ServiceInstance> getInstances(String serviceId) {
		try {
			return Optional.of(serviceDiscovery.getInstances(serviceId)).map(instances -> {
						ServiceCache.setInstances(serviceId, instances);
						return instances;
					}).get();
		}
		catch (Exception e) {
			if (failureToleranceEnabled) {
				return ServiceCache.getInstances(serviceId);
			}
			throw new RuntimeException(
					"Can not get hosts from nacos server. serviceId: " + serviceId, e);
		}
	}
	// ...
}
```

接下来看com.alibaba.cloud.nacos.discovery.NacosServiceDiscovery

```java
public class NacosServiceDiscovery {
        // ...
        public List<ServiceInstance> getInstances(String serviceId) throws NacosException {
		String group = discoveryProperties.getGroup();
		List<Instance> instances = namingService().selectInstances(serviceId, group,
				true);
		return hostToServiceInstanceList(instances, serviceId);
	}
	// ...
	private NamingService namingService() {
		return nacosServiceManager
				.getNamingService(discoveryProperties.getNacosProperties());
	}
}
```

再看com.alibaba.nacos.client.naming.NacosNamingService

```java
@SuppressWarnings("PMD.ServiceOrDaoClassShouldEndWithImplRule")
public class NacosNamingService implements NamingService {
    // ...
    @Override
    public List<Instance> selectInstances(String serviceName, String groupName, boolean healthy) throws NacosException {
        return selectInstances(serviceName, groupName, healthy, true);
    }
    // ...
    @Override
    public List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy,
            boolean subscribe) throws NacosException {
        
        ServiceInfo serviceInfo;
        if (subscribe) { // 走到这个分支
            serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),
                    StringUtils.join(clusters, ","));
        } else {
            serviceInfo = hostReactor
                    .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),
                            StringUtils.join(clusters, ","));
        }
        return selectInstances(serviceInfo, healthy);
    }
    // ...
}

```

再看com.alibaba.nacos.client.naming.core.HostReactor

```java
public class HostReactor implements Closeable {
    // ...
    public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
        
        NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
        String key = ServiceInfo.getKey(serviceName, clusters);
        if (failoverReactor.isFailoverSwitch()) {
            return failoverReactor.getService(key);
        }
        
        ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);
        
        if (null == serviceObj) {
            serviceObj = new ServiceInfo(serviceName, clusters);
            
            serviceInfoMap.put(serviceObj.getKey(), serviceObj);
            
            updatingMap.put(serviceName, new Object());
            updateServiceNow(serviceName, clusters);
            updatingMap.remove(serviceName);
            
        } else if (updatingMap.containsKey(serviceName)) {
            
            if (UPDATE_HOLD_INTERVAL > 0) {
                // hold a moment waiting for update finish
                synchronized (serviceObj) {
                    try {
                        serviceObj.wait(UPDATE_HOLD_INTERVAL);
                    } catch (InterruptedException e) {
                        NAMING_LOGGER
                                .error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                    }
                }
            }
        }
        
        scheduleUpdateIfAbsent(serviceName, clusters); // 定时更新
        
        return serviceInfoMap.get(serviceObj.getKey());
    }
    // ...
    private ServiceInfo getServiceInfo0(String serviceName, String clusters) {
        
        String key = ServiceInfo.getKey(serviceName, clusters);
        
        return serviceInfoMap.get(key);
    }
    // ...
    public void scheduleUpdateIfAbsent(String serviceName, String clusters) {
        if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
            return;
        }
        
        synchronized (futureMap) {
            if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
                return;
            }
            
            ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, clusters));
            futureMap.put(ServiceInfo.getKey(serviceName, clusters), future);
        }
    }
    // ...
    public synchronized ScheduledFuture<?> addTask(UpdateTask task) {
        return executor.schedule(task, DEFAULT_DELAY, TimeUnit.MILLISECONDS);
    }
    
    
    public HostReactor(NamingProxy serverProxy, BeatReactor beatReactor, String cacheDir) {
        this(serverProxy, beatReactor, cacheDir, false, false, UtilAndComs.DEFAULT_POLLING_THREAD_COUNT);
    }
    
    public HostReactor(NamingProxy serverProxy, BeatReactor beatReactor, String cacheDir, boolean loadCacheAtStart,
            boolean pushEmptyProtection, int pollingThreadCount) {
        // init executorService
        this.executor = new ScheduledThreadPoolExecutor(pollingThreadCount, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setDaemon(true);
                thread.setName("com.alibaba.nacos.client.naming.updater");
                return thread;
            }
        });
        // ...
    }
}
```

从这里可以看到，调用方的提供方实例列表是异步更新的，有一定的时间差，一次存在这种可能。

### 解决

如上分析，流量基本上是因为nacos客户端更新缓存延迟造成的，如果要解决的，可以提前从nacos上下掉要重启的实例，等待一段时间后再重启

因此，可以在应用上提供一个自动下线服务的接口，在发布之前curl一下，等待一段时间后再重启应用

自动下线服务业很简单，直接利用spring cloud 提供的组件即可

参考代码如下

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.serviceregistry.Registration;
import org.springframework.cloud.client.serviceregistry.ServiceRegistry;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DeregisterController {

    @Autowired
    private Registration registration;

    @Autowired
    private ServiceRegistry serviceRegistry;

    @RequestMapping("/deregister")
    public String deregister() {
        serviceRegistry.deregister(registration);
        return "ok";
    }
}

```
