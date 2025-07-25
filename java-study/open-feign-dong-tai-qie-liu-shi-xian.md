# Open feign动态切流实现

## 背景

最近在做服务拆分，涉及到feign接口迁移，因为调用方比较多，希望可以有一些其他的方案，在调用方不做代码修改的情况下，也可以实现流量迁移，同时可以控制切流的节奏

## 目标

1、运行时做feign调用切流

2、切流比例可以动态控制

## 方案

上文，我们做过Open Feign源码分析（感兴趣的同学可以出门左转查看上一篇文章《Open feign源码分析》），可以得知Open feign接口在调用时：最终会生成RequestTemplate，通过操作RequestTemplate发送http请求，解析http响应；同时Open feign提供了运行时增强的入口，通过RequestIntercepter实现

<pre class="language-java"><code class="lang-java"><strong>public interface RequestInterceptor {    
</strong>    void apply(RequestTemplate template);
}
</code></pre>

接下来，我们查看RequestTemplate

```java
public final class RequestTemplate implements Serializable {

  private static final Pattern QUERY_STRING_PATTERN = Pattern.compile("(?<!\\{)\\?");
  private final Map<String, QueryTemplate> queries = new LinkedHashMap<>();
  private final Map<String, HeaderTemplate> headers = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
  private String target;
  private String fragment;
  private boolean resolved = false;
  private UriTemplate uriTemplate;
  private BodyTemplate bodyTemplate;
  private HttpMethod method;
  private transient Charset charset = Util.UTF_8;
  private Request.Body body = Request.Body.empty();
  private boolean decodeSlash = true;
  private CollectionFormat collectionFormat = CollectionFormat.EXPLODED;
  private MethodMetadata methodMetadata;
  private Target<?> feignTarget;
  ...
}
```

其中MethodMetadata属性记录了原始Open feign 接口的方法

那如何做动态切流呢？

**假设提供服务的新老服务serviceA和serviceB有以下特点**

**1、接口path都相同**

**2、接口入参格式相同**

**3、接口出参格式相同**

**那么，我们只需要在调用serviceA时，将地址换成serviceB的地址，即可实现流量打到serviceB，且不出现数据解析异常**

基于以上分析，我们可以在迁移到新服务时，参考上面3点要求做实现

接下来的问题就是，如何在调用serviceA时将地址换成serviceB的地址，在java体系中，我们很容易想到的就是AOP，那么如何实现呢？

首先，我们先定义一个注解，注解中包含了新服务的地址，以及切流的配置

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ForwardTraffic {

  /**
   * 新服务地址
   *
   * @return
   */
  String targetService() default "";

  /**
   * 切流百分比配置key
   *
   * @return
   */
  String rateKey() default "";
}
```

从注解定义，我们可以看到切流控制可以精确到方法级别

接下来，我们定义一个类，用于从配置中心后去切流的配置

```java
@Data
@RefreshScope
public class ForwardTrafficProperties {

  public static final int MAX_RATE = 10000;

  public static final int MIN_RATE = 0;

  /**
   * 全局开关
   */
  private Boolean enabled = true;

  private Map<String, Integer> rateMap = new HashMap<>();
}
```

然后，我们在定义一个接口，用于判断是否切流

```java
public interface ForwardTrafficSwitch {

  /**
   * 根据key判断是否切流
   *
   * @param key
   * @return
   */
  boolean forward(String key);

}
```

接着，我们定义一个默认实现

```java
public class DefaultForwardTrafficSwitch implements ForwardTrafficSwitch {
  // 切流比例配置
  private ForwardTrafficProperties forwardTrafficProperties;

  public DefaultForwardTrafficSwitch(ForwardTrafficProperties forwardTrafficProperties) {
    this.forwardTrafficProperties = forwardTrafficProperties;
  }

  @Override
  public boolean forward(String key) {
    log.debug("DefaultForwardTrafficSwitch.forward, key={}, forwardTrafficProperties={}", key, forwardTrafficProperties);
    if (forwardTrafficProperties == null) {
      return false;
    }
    if (!Boolean.TRUE.equals(forwardTrafficProperties.getEnabled())) {
      if (log.isDebugEnabled()) {
        log.debug("DefaultForwardTrafficSwitch.forward, forward traffic disabled, ignore key: {}", key);
      }
      return false;
    }

    Integer rate = Optional.ofNullable(forwardTrafficProperties.getRateMap())
        .orElse(new HashMap<>())
        .get(key);
    if (rate == null) {
      if (log.isDebugEnabled()) {
        log.debug("DefaultForwardTrafficSwitch.forward, no rate for key: {}, ignore", key);
      }
      return false;
    }

    if (rate < ForwardTrafficProperties.MIN_RATE) {
      rate = ForwardTrafficProperties.MIN_RATE;
    }
    
    if (rate > ForwardTrafficProperties.MAX_RATE) {
      rate = ForwardTrafficProperties.MAX_RATE;
    }
    // 按照比例切，阈值是10000
    boolean result = ThreadLocalRandom.current().nextInt(ForwardTrafficProperties.MAX_RATE) < rate;
    if (log.isDebugEnabled()) {
      log.debug("DefaultForwardTrafficSwitch.forward, key: {}, rate: {}, result: {}", key, rate, result);
    }
    return result;
  }
}
```

接下来，我们定义注解解析类

```java
@Slf4j
public class ForwardTrafficClassResolver {

  /**
   * 缓存方法对应的数据源
   */
  private final Map<Object, AnnotationAttributes> forwardCache = new ConcurrentHashMap<>();

  private final boolean allowedPublicOnly;

  /**
   * 加入扩展, 给外部一个修改aop条件的机会
   *
   * @param allowedPublicOnly 只允许公共的方法, 默认为true
   */
  public ForwardTrafficClassResolver(boolean allowedPublicOnly) {
    this.allowedPublicOnly = allowedPublicOnly;
  }

  /**
   * 获取目标service
   *
   * @param method
   * @param targetClass
   * @return
   */
  public String getTargetService(Method method, Class<?> targetClass) {
    return findAttribute(method, targetClass, "targetService");
  }

  /**
   * 获取切流比例
   *
   * @param method
   * @param targetClass
   * @return
   */
  public String getRateKey(Method method, Class<?> targetClass) {
    return findAttribute(method, targetClass, "rateKey");
  }

  /**
   * 从缓存获取数据
   *
   * @param method      方法
   * @param targetClass 目标对象
   * @return
   */
  private String findAttribute(Method method, Class<?> targetClass, String key) {
    if (method.getDeclaringClass() == Object.class) {
      return null;
    }

    Object cacheKey = new MethodClassKey(method, targetClass);
    AnnotationAttributes attributes = this.forwardCache.get(cacheKey);
    if (attributes == null) {
      attributes = computeAttributes(method, targetClass);
      if (attributes == null) {
        return null;
      }

      this.forwardCache.put(cacheKey, attributes);
    }

    return attributes.getString(key);
  }

  /**
   * 查找注解的顺序
   * 1. 当前方法
   * 2. 桥接方法
   * 3. 当前类开始一直找到Object
   *
   * @param method      方法
   * @param targetClass 目标对象
   * @return
   */
  private AnnotationAttributes computeAttributes(Method method, Class<?> targetClass) {
    if (allowedPublicOnly && !Modifier.isPublic(method.getModifiers())) {
      return null;
    }

    //1. 从当前方法接口中获取
    AnnotationAttributes attributes = findForwardTrafficAttribute(method);
    if (attributes != null) {
      return attributes;
    }

    Class<?> userClass = ClassUtils.getUserClass(targetClass);
    // JDK代理时,  获取实现类的方法声明.  method: 接口的方法, specificMethod: 实现类方法
    Method specificMethod = ClassUtils.getMostSpecificMethod(method, userClass);

    specificMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
    //2. 从桥接方法查找
    attributes = findForwardTrafficAttribute(specificMethod);
    if (attributes != null) {
      return attributes;
    }

    // 从当前方法声明的类查找
    attributes = findForwardTrafficAttribute(userClass);
    if (attributes != null && ClassUtils.isUserLevelMethod(method)) {
      return attributes;
    }

    //since 3.4.1 从接口查找，只取第一个找到的
    for (Class<?> interfaceClazz : ClassUtils.getAllInterfacesForClassAsSet(userClass)) {
      attributes = findForwardTrafficAttribute(interfaceClazz);
      if (attributes != null) {
        return attributes;
      }
    }

    // 如果存在桥接方法
    if (specificMethod != method) {
      // 从桥接方法查找
      attributes = findForwardTrafficAttribute(method);
      if (attributes != null) {
        return attributes;
      }
      // 从桥接方法声明的类查找
      attributes = findForwardTrafficAttribute(method.getDeclaringClass());
      if (attributes != null && ClassUtils.isUserLevelMethod(method)) {
        return attributes;
      }
    }

    return null;
  }


  /**
   * 通过 AnnotatedElement 查找标记的注解
   *
   * @param ae AnnotatedElement
   * @return
   */
  private AnnotationAttributes findForwardTrafficAttribute(AnnotatedElement ae) {
    return AnnotatedElementUtils.getMergedAnnotationAttributes(ae, ForwardTraffic.class);
  }
}

```

这个类的主要逻辑就是从方法或者接口申明上去找@ForwardTraffic注解，找到之后，获取到注解的targetService和rateKey属性

再接下来，我们定义拦截器

```java
@Slf4j
public class ForwardTrafficInterceptor implements RequestInterceptor, EnvironmentAware, BeanDefinitionRegistryPostProcessor {

  private ForwardTrafficSwitch forwardTrafficSwitch;

  private final ForwardTrafficClassResolver forwardTrafficClassResolver;

  private Environment environment;

  private ConfigurableListableBeanFactory beanFactory;

  public ForwardTrafficInterceptor(ForwardTrafficSwitch forwardTrafficSwitch) {
    this.forwardTrafficSwitch = forwardTrafficSwitch;
    this.forwardTrafficClassResolver = new ForwardTrafficClassResolver(true);
  }

  @Override
  public void setEnvironment(Environment environment) {
    this.environment = environment;
  }

  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;

  }

  /**
   * 参考FeignClientsRegistrar
   * 环境变量替换
   *
   * @param value
   * @return
   */
  private String resolve(String value) {
    if (StringUtils.hasText(value)) {
      if (beanFactory == null) {
        return this.environment.resolvePlaceholders(value);
      }
      BeanExpressionResolver resolver = beanFactory.getBeanExpressionResolver();
      String resolved = beanFactory.resolveEmbeddedValue(value);
      if (resolver == null) {
        return resolved;
      }
      Object evaluateValue = resolver.evaluate(resolved, new BeanExpressionContext(beanFactory, null));
      if (evaluateValue != null) {
        return String.valueOf(evaluateValue);
      }
      return null;
    }
    return value;
  }

  @Override
  public void apply(RequestTemplate template) {
    MethodMetadata methodMetadata = template.methodMetadata();
    String targetService = forwardTrafficClassResolver.getTargetService(methodMetadata.method(), methodMetadata.targetType());
    if (StrUtil.isBlank(targetService)) {
      return;
    }

    String rateKey = forwardTrafficClassResolver.getRateKey(methodMetadata.method(), methodMetadata.targetType());
    if (StrUtil.isBlank(rateKey)) {
      return;
    }

    if (!forwardTrafficSwitch.forward(rateKey)) {
      return;
    }

    String originUrl = template.feignTarget().url();
    String newService = resolve(targetService);
    try {
      String url = replaceHost(originUrl, newService);
      if (log.isDebugEnabled()) {
        log.debug("Forward traffic hit path={}, originUrl={}, newUrl={}", template.path(), originUrl, url);
      }

      template.target(url);
    } catch (MalformedURLException e) {
      log.error("Forward traffic error path={}, originUrl={}, newService={}", template.path(), originUrl, newService, e);
    }
  }

  private String replaceHost(String originUrl, String host) throws MalformedURLException {
    if (StrUtil.isBlank(host)) {
      return originUrl;
    }

    URL url = new URL(originUrl);
    if (StrUtil.startWith(host, "http")) {
      return new URL(url.getProtocol(), new URL(host).getHost(), url.getPort(), url.getFile()).toString();
    }

    return new URL(url.getProtocol(), host, url.getPort(), url.getFile()).toString();
  }
}

```

我们重点看apply方法，其主要逻辑如下

1、通过RequestTemplate.methodMetadata去查找如是否被@ForwardTraffic标记，如果被标记，则返回对应的targetService和rateKey

2、如果没有被标记，也就是targetService和rateKey为空，此时不做任何操作

3、根据rateKey判断是否需要切流（通过比例控制），如果未命中，此时不做任何操作

4、如果命中，则替换地址，也就是replaceHost方法

这里需要注意，因为我们通常不希望将地址写死，因此这里参考open feign原始实现，copy了resolve方法，支持通过环境变量和配置来动态解析目标地址，也就是说支持targetService是一个spring el表达式

至此，我们只需要在feign client定义的方法上加上对应的@ForwardTraffic注解，调用方升级下版本即可实现运行时切流了。

**到这里，我们的目标完成了大半；为什么是大半？因为大型项目中，通常调用方不一定全部是java，也有可能是go或者python，因此可能还是会有其他流量打到旧的服务上。**

因此我们还需要在旧的服务上实现流量的转发逻辑

那么如何实现呢？

还记得上面我们说的3个假设吗？

1、新老服务path一样

2、接口入参格式一样

3、接口出参格式一样

因此，我们可以有几种方式来做切流

**基于网关的切流**

假设内部调用先通过网关，然后再到目标服务，那么我们可以在网关层面直接做流量的转发，不论是基于nginx的流量转发还是基于shenyu的流量转发都可以做到按照比例做流量切分，这里不做详细介绍

**基于目标服务的切流**

在客户端控制的部分，我们已经定义了注解以及切流的开关控制类，那么我们是否可以复用这一部分呢？假设可以复用这部分，我们做一个Contoroller层面的AOP不就可以完成服务端的切流了吗

那如何复用呢？

对于切流开关控制类，比较容易，只需要我们引入对应的bean即可

对于注解我们应该如何复用呢？

答案也很简单，我们只需要在controller层面声明实现了某个Feign Client，我们即可将对应的controller方法和feign 接口方法建立映射关系，且不会影响到现有controller逻辑，如下所示

```java
@FeignClient(name = "XxxApi", url = PLACE_HOLD_SERVICE_NAME, path = AgencyApi.PATH)
public interface XxxApi {

  String PATH = "/api/xxx";
  
  @ForwardTraffic(targetService="${newXxxA.host}", rateKey="XxxApi.get")
  @GetMapping("/get")
  Object get(@RequestParam("id") Long id);
}

@RestController
@RequestMapping(path = XxxApi.PATH)
public class XxxAApiImpl implements XxxApi {

  @Override
  public Object get(Long agencyId) {
    // ...
  }
```

好，有了映射关系，我们即可通过AOP来做切流。我们需要对所有的controller方法做AOP吗？答案是没必要，我们只需要对需要做切流的方法做AOP即可

因此，我们需要先定义一个注解，然后在根据注解实现AOP

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ControllerForward {

}
```

接下来，我们先实现拦截器的逻辑

```java


@Slf4j
public class ControllerForwardAnnotationInterceptor implements MethodInterceptor, ApplicationContextAware, EnvironmentAware, BeanDefinitionRegistryPostProcessor {

  private final Map<String, Map<Class<?>, Object>> FEIGN_CACHE= new ConcurrentHashMap<>();

  private ForwardTrafficProperties forwardTrafficProperties;

  private ForwardTrafficSwitch forwardTrafficSwitch;

  private ForwardTrafficClassResolver resolver = new ForwardTrafficClassResolver(true);

  private Environment environment;

  private ConfigurableListableBeanFactory beanFactory;

  private FeignClientBuilder feignClientBuilder;

  public ControllerForwardAnnotationInterceptor(ForwardTrafficProperties forwardTrafficProperties, ForwardTrafficSwitch forwardTrafficSwitch) {
    this.forwardTrafficProperties = forwardTrafficProperties;
    this.forwardTrafficSwitch = forwardTrafficSwitch;
  }

  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    this.feignClientBuilder = new FeignClientBuilder(applicationContext);
  }

  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
  }

  @Override
  public void setEnvironment(Environment environment) {
    this.environment = environment;
  }

  @Nullable
  @Override
  public Object invoke(@NotNull MethodInvocation invocation) throws Throwable {
    if (!Boolean.TRUE.equals(forwardTrafficProperties.getEnabled())) {
      return invocation.proceed();
    }

    Class<?> feignClientClass = getFeignClientClass(invocation.getMethod());
    if (feignClientClass == null) {
      return invocation.proceed();
    }

    Method method;
    try {
      method = feignClientClass.getMethod(invocation.getMethod().getName(), invocation.getMethod().getParameterTypes());
    } catch (NoSuchMethodException | SecurityException e) {
      return invocation.proceed();
    }

    String targetService = resolver.getTargetService(method, feignClientClass);
    if (StrUtil.isBlank(targetService)) {
      return invocation.proceed();
    }

    String rateKey = resolver.getRateKey(method, feignClientClass);
    if (StrUtil.isBlank(rateKey)) {
      return invocation.proceed();
    }

    if (!forwardTrafficSwitch.forward(rateKey)) {
      return invocation.proceed();
    }

    if (log.isDebugEnabled()) {
      log.debug("controller forward traffic: {}.{} target:{}", method.getDeclaringClass().getName(), method.getName(), resolve(targetService));
    }
    Object feignClient = getFeignClient(feignClientClass, targetService);
    if (feignClient == null) {
      return invocation.proceed();
    }

    return method.invoke(feignClient, invocation.getArguments());
  }


  private Class<?> getFeignClientClass(Method method) {
    try {
      Class<?>[] ifaces = method.getDeclaringClass().getInterfaces();
      if (ifaces == null || ifaces.length == 0) {
        return null;
      }
      for (Class<?> iface : ifaces) {
        if (iface.isAnnotationPresent(FeignClient.class)) {
          return iface;
        }
      }
    } catch (Exception e) {
      log.error("getFeignClientClass Exception", e);
    }

    return null;
  }

  public Object getFeignClient(Class<?> feignClientClass, String targetService) {
    if (feignClientClass == null || StrUtil.isBlank(targetService)) {
      throw new BizException("feignClientClass or targetService is null");
    }

    Map<Class<?>, Object> feignCache = FEIGN_CACHE.computeIfAbsent(targetService, k -> new ConcurrentHashMap<>());
    return feignCache.computeIfAbsent(feignClientClass, key -> buildFeignClient(feignClientClass, targetService));
  }

  private <T> T buildFeignClient(Class<T> feignClientClass, String targetService) {
    AnnotationAttributes aa = AnnotatedElementUtils.getMergedAnnotationAttributes(feignClientClass, FeignClient.class);
    if (aa == null) {
      return null;
    }

    return this.feignClientBuilder.forType(feignClientClass, "forward" + aa.getString("name"))
        .path(aa.getString("path"))
        .url(resolve(targetService))
        .build();
  }

  private String resolve(String value) {
    if (StringUtils.hasText(value)) {
      if (beanFactory == null) {
        return this.environment.resolvePlaceholders(value);
      }
      BeanExpressionResolver resolver = beanFactory.getBeanExpressionResolver();
      String resolved = beanFactory.resolveEmbeddedValue(value);
      if (resolver == null) {
        return resolved;
      }
      Object evaluateValue = resolver.evaluate(resolved, new BeanExpressionContext(beanFactory, null));
      if (evaluateValue != null) {
        return String.valueOf(evaluateValue);
      }
      return null;
    }
    return value;
  }
}

```

拦截器实现了MethodIntercepter，我们重点看下invoke方法，主要逻辑如下

1、通过method获取到对应的Feign client class定义

2、通过反射获取到Feign Client 方法上的@ForwardTraffic注解，并获取targetService和rateKey属性

3、根据rateKey判断是否需要切流

4、如果需要切流，根据targetService生成一个新的feign client

5、通过反射调用新的feign client（简化流量转发逻辑，将新的http调用代理给feign client）

对于步骤4，是一个小知识点，感兴趣的同学可以出门左转（《spring cloud手动创建feign client》）

接下来，我们需要定义好切面

```java
/**
 * 参考dynamic datasource
 *
 * @see com.baomidou.dynamic.datasource.aop.DynamicDataSourceAnnotationAdvisor
 */
public class ControllerForwardAnnotationAdvisor extends AbstractPointcutAdvisor implements BeanFactoryAware {

  private final Advice advice;

  private final Class<? extends Annotation> annotation;

  private final Pointcut pointcut;

  public ControllerForwardAnnotationAdvisor(Advice advice, Class<? extends Annotation> annotation) {
    this.advice = advice;
    this.annotation = annotation;
    this.pointcut = buildPointcut();
  }

  @Override
  public Pointcut getPointcut() {
    return this.pointcut;
  }

  @Override
  public Advice getAdvice() {
    return this.advice;
  }

  @Override
  public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
    if (this.advice instanceof BeanFactoryAware) {
      ((BeanFactoryAware) this.advice).setBeanFactory(beanFactory);
    }
  }

  private Pointcut buildPointcut() {
    Pointcut cpc = new AnnotationMatchingPointcut(annotation, true);
    Pointcut mpc = new ControllerForwardAnnotationAdvisor.AnnotationMethodPoint(annotation);
    return new ComposablePointcut(cpc).union(mpc);
  }

  /**
   * In order to be compatible with the spring lower than 5.0
   */
  private static class AnnotationMethodPoint implements Pointcut {

    private final Class<? extends Annotation> annotationType;

    public AnnotationMethodPoint(Class<? extends Annotation> annotationType) {
      Assert.notNull(annotationType, "Annotation type must not be null");
      this.annotationType = annotationType;
    }

    @Override
    public ClassFilter getClassFilter() {
      return ClassFilter.TRUE;
    }

    @Override
    public MethodMatcher getMethodMatcher() {
      return new ControllerForwardAnnotationAdvisor.AnnotationMethodPoint.AnnotationMethodMatcher(annotationType);
    }

    private static class AnnotationMethodMatcher extends StaticMethodMatcher {

      private final Class<? extends Annotation> annotationType;

      public AnnotationMethodMatcher(Class<? extends Annotation> annotationType) {
        this.annotationType = annotationType;
      }

      @Override
      public boolean matches(Method method, Class<?> targetClass) {
        if (matchesMethod(method)) {
          return true;
        }
        // Proxy classes never have annotations on their redeclared methods.
        if (Proxy.isProxyClass(targetClass)) {
          return false;
        }
        // The method may be on an interface, so let's check on the target class as well.
        Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);
        return (specificMethod != method && matchesMethod(specificMethod));
      }

      private boolean matchesMethod(Method method) {
        return AnnotatedElementUtils.hasAnnotation(method, this.annotationType);
      }
    }
  }
}

```

这里不做过多介绍，知名的开源组件都有类似的实现，大同小异

最后做一个自动配置类即可

```java
@Configuration
public class ControllerForwardAutoConfiguration {

  @Bean
  public ControllerForwardAnnotationInterceptor controllerForwardAnnotationInterceptor(ForwardTrafficProperties forwardTrafficProperties,
      ForwardTrafficSwitch forwardTrafficSwitch) {
    return new ControllerForwardAnnotationInterceptor(forwardTrafficProperties, forwardTrafficSwitch);
  }

  @Bean
  public Advisor controllerForwardAnnotationAdvisor(ControllerForwardAnnotationInterceptor interceptor) {
    ControllerForwardAnnotationAdvisor advisor = new ControllerForwardAnnotationAdvisor(interceptor, ControllerForward.class);
    return advisor;
  }
}
```

到这里，我们就大功告成了，感兴趣的同学可以试试
