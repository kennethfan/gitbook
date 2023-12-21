---
description: spring cloud自定义loaderbalancer实现流量就近访问
---

# spring cloud自定义loaderbalancer实现流量就近访问

## 背景 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-bei-jing" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-bei-jing"></a>

文旅项目处于稳定需要，目前每个服务部署了多个实例，分布在不同的结点；同时为了应对甲方的稳定性需要，接下来会上云

这样，每个服务的多个节点会跨网络，甚至是不同的运营商，这样会带来一些问题

基于现有的负载均衡策略，会出现跨网络节点互相访问，这样带来的超时会比较明显，因此需要解决这个问题

## 目标 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-mu-biao" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-mu-biao"></a>

实现流量的就近访问

## 方案 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-fang-an" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-fang-an"></a>

基于spring cloud loadbalancer

### 代码分析 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-dai-ma-fen-xi" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-dai-ma-fen-xi"></a>

先看spring-cloud-loadbalancer下spring.factories

```
# AutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.loadbalancer.config.LoadBalancerAutoConfiguration,\
org.springframework.cloud.loadbalancer.config.BlockingLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.loadbalancer.config.LoadBalancerCacheAutoConfiguration,\
org.springframework.cloud.loadbalancer.security.OAuth2LoadBalancerClientAutoConfiguration,\
org.springframework.cloud.loadbalancer.config.LoadBalancerStatsAutoConfiguration
```

重点关注org.springframework.cloud.loadbalancer.config.LoadBalancerAutoConfiguration

```java
@Configuration(
    proxyBeanMethods = false
)
@LoadBalancerClients // 重点看这里
@EnableConfigurationProperties({LoadBalancerClientsProperties.class})
@AutoConfigureBefore({ReactorLoadBalancerClientAutoConfiguration.class, LoadBalancerBeanPostProcessorAutoConfiguration.class})
@ConditionalOnProperty(
    value = {"spring.cloud.loadbalancer.enabled"},
    havingValue = "true",
    matchIfMissing = true
)
public class LoadBalancerAutoConfiguration {
}
```

接着看org.springframework.cloud.loadbalancer.annotation.LoadBalancerClients

```java
@Configuration(
    proxyBeanMethods = false
)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({LoadBalancerClientConfigurationRegistrar.class})
public @interface LoadBalancerClients {
    LoadBalancerClient[] value() default {};
 
    Class<?>[] defaultConfiguration() default {};
}
```

通过defaultConfiguration可以指定默认配置，如果不指定，会用到org.springframework.cloud.loadbalancer.annotation.LoadBalancerClientConfiguration

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnDiscoveryEnabled
public class LoadBalancerClientConfiguration {
    private static final int REACTIVE_SERVICE_INSTANCE_SUPPLIER_ORDER = 193827465;
 
    public LoadBalancerClientConfiguration() {
    }
 
    @Bean
    @ConditionalOnMissingBean  // 默认走这里，实际走的是轮询
    public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(Environment environment, LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty("loadbalancer.client.name");
        return new RoundRobinLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

为什么默认会走到org.springframework.cloud.loadbalancer.annotation.LoadBalancerClientConfiguration

看org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory

```java
public class LoadBalancerClientFactory extends NamedContextFactory<LoadBalancerClientSpecification> implements Factory<ServiceInstance> {
    private static final Log log = LogFactory.getLog(LoadBalancerClientFactory.class);
    public static final String NAMESPACE = "loadbalancer";
    public static final String PROPERTY_NAME = "loadbalancer.client.name";
    private final LoadBalancerClientsProperties properties;
 
    /** @deprecated */
    @Deprecated
    public LoadBalancerClientFactory() {
        this((LoadBalancerClientsProperties)null);
    }
 
    public LoadBalancerClientFactory(LoadBalancerClientsProperties properties) {
        super(LoadBalancerClientConfiguration.class, "loadbalancer", "loadbalancer.client.name"); // 主要是这里
        this.properties = properties;
    }
    // ...
}
```

如果还想继续往下跟，可以看org.springframework.cloud.gateway.filter.ReactiveLoadBalancerClientFilter

```java
public class ReactiveLoadBalancerClientFilter implements GlobalFilter, Ordered {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = (URI)exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = (String)exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);
        if (url != null && ("lb".equals(url.getScheme()) || "lb".equals(schemePrefix))) {
            ServerWebExchangeUtils.addOriginalRequestUrl(exchange, url);
            if (log.isTraceEnabled()) {
                log.trace(ReactiveLoadBalancerClientFilter.class.getSimpleName() + " url before: " + url);
            }
 
            URI requestUri = (URI)exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
            String serviceId = requestUri.getHost();
            Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator.getSupportedLifecycleProcessors(this.clientFactory.getInstances(serviceId, LoadBalancerLifecycle.class), RequestDataContext.class, ResponseData.class, ServiceInstance.class);   // 重点在这里
            DefaultRequest<RequestDataContext> lbRequest = new DefaultRequest(new RequestDataContext(new RequestData(exchange.getRequest()), this.getHint(serviceId)));
            // ...
    }
    
    // ...
 
    private Mono<Response<ServiceInstance>> choose(Request<RequestDataContext> lbRequest, String serviceId, Set<LoadBalancerLifecycle> supportedLifecycleProcessors) {
        ReactorLoadBalancer<ServiceInstance> loadBalancer = (ReactorLoadBalancer)this.clientFactory.getInstance(serviceId, ReactorServiceInstanceLoadBalancer.class);  // 以及这里
        if (loadBalancer == null) {
            throw new NotFoundException("No loadbalancer available for " + serviceId);
        } else {
            supportedLifecycleProcessors.forEach((lifecycle) -> {
                lifecycle.onStart(lbRequest);
            });
            return loadBalancer.choose(lbRequest);
        }
     }
     // ...
}
```

### 具体实施 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-ju-ti-shi-shi" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-ju-ti-shi-shi"></a>

替换默认配置

```java
@Slf4j
@ConditionalOnDiscoveryEnabled
public class NearbyLoadBalancerConfig {
 
    public NearbyLoadBalancerConfig() {
    }
 
    @PostConstruct
    public void init() {
        log.info("NearbyLoadBalancerConfig inited");
    }
 
    @Bean
    @ConditionalOnMissingBean
    public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(LoadBalancerClientFactory loadBalancerClientFactory, Environment environment, InetUtils inetUtils) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new NearbyServiceLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name, environment, inetUtils);
    }
}
```

应用端通过@LoadBalancerClients 注解修改默认配置

```java
@LoadBalancerClients(defaultConfiguration = NearbyLoadBalancerConfig.class)
@SpringBootApplication
public class Application {
 
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        log.info("......服务启动成功!");
    }
 
}
```

### 负载策略 <a href="#springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-fu-zai-ce-lve" id="springcloud-zi-ding-yi-loaderbalancer-shi-xian-liu-liang-jiu-jin-fang-wen-fu-zai-ce-lve"></a>

1、优先访问通ip节点，也可以访问同网段节点（此处未做实现）

2、如果没有通ip节点，轮询选择一个

3、开关控制，关闭之后继续走轮询

代码如下

```java
@Slf4j
public class NearbyServiceLoadBalancer implements ReactorServiceInstanceLoadBalancer {
 
    public static final String SWITCH_KEY = "discovery.nearby.enable"; // 开关配置key
  
    private ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final String serviceId;
    private Environment environment;
 
    private final AtomicInteger position;
    private final String nearbyIp; // 本机ip
 
    private RoundRobinLoadBalancer roundRobinLoadBalancer;
 
 
    public NearbyServiceLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider, String serviceId, Environment environment, InetUtils inetUtils) {
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.serviceId = serviceId;
        this.environment = environment;
 
        this.position = new AtomicInteger(1000);
        this.nearbyIp = inetUtils.findFirstNonLoopbackHostInfo().getIpAddress();
 
        this.roundRobinLoadBalancer = new RoundRobinLoadBalancer(serviceInstanceListSupplierProvider, serviceId);
 
        log.info("NearbyServiceLoadBalancer.construct, serviceId={}, nearbyIp={}", this.serviceId, this.nearbyIp);
    }
 
    /**
     * 判断是否走就近访问策略
     **/
    private boolean disableNearby() {
        return StringUtils.isBlank(this.nearbyIp)
                || !Boolean.TRUE.toString().equals(environment.getProperty(SWITCH_KEY));
    }
 
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        if (disableNearby()) {
            return this.roundRobinLoadBalancer.choose(request);
        }
 
        ServiceInstanceListSupplier supplier = this.serviceInstanceListSupplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next().map((serviceInstances) -> this.processInstanceResponse(supplier, serviceInstances));
    }
 
    protected Response<ServiceInstance> processInstanceResponse(ServiceInstanceListSupplier supplier, List<ServiceInstance> serviceInstances) {
        Response<ServiceInstance> serviceInstanceResponse = this.getInstanceResponse(serviceInstances);
        if (supplier instanceof SelectedInstanceCallback && serviceInstanceResponse.hasServer()) {
            ((SelectedInstanceCallback) supplier).selectedServiceInstance(serviceInstanceResponse.getServer());
        }
 
        return serviceInstanceResponse;
    }
 
    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        ServiceInstance instance = selectInstance(instances);
        return instance == null ? new EmptyResponse() : new DefaultResponse(instance);
    }
 
    /**
     * 选择最合适的节点
     **/
    protected ServiceInstance selectInstance(List<ServiceInstance> instances) {
        if (log.isDebugEnabled()) {
            log.debug("NearbyServiceLoadBalancer.selectInstance, instances={}", formatServiceInstances(instances));
        }
 
        if (CollectionUtils.isEmpty(instances)) {
            return null;
        }
 
        // 只有一个候选节点，不用走策略，直接使用
        if (instances.size() == 1) {
            return instances.get(0);
        }
 
        ServiceInstance instance = instances.stream()
                .filter(o -> this.nearbyIp.equals(o.getHost()))
                .findFirst()
                .orElse(null);
        if (instance == null) {
            int pos = Math.abs(this.position.incrementAndGet());
            instance = instances.get(pos % instances.size());
        }
 
        if (log.isDebugEnabled()) {
            log.debug("NearbyServiceLoadBalancer.selectInstance, instance={}", formatServiceInstance(instance));
        }
 
        return instance;
    }
 
    private String formatServiceInstances(List<ServiceInstance> serviceInstances) {
        if (serviceInstances == null) {
            return null;
        }
 
        if (serviceInstances.size() == 0) {
            return "[]";
        }
 
        return "[" + serviceInstances.stream().map(this::formatServiceInstance).reduce((s1, s2) -> s1 + "," + s2).get() + "]";
    }
 
    private String formatServiceInstance(ServiceInstance serviceInstance) {
        if (serviceInstance == null) {
            return null;
        }
 
        return "{instanceId=" + serviceInstance.getInstanceId() + "}";
    }
}
```

### 功能开启

```yaml
discovery:
  nearby:
    enable: true
```
