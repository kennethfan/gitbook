# Open feign源码分析

## 前沿

最近在做服务拆分和迁移，其中涉及到一些feign接口的迁移，因为涉及到上游比较多，希望可以尽量减少上游的改动量，因此决定调研下open feign

## 正文

### 入口EnableFeignClients

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

	/**
	 * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
	 * declarations e.g.: {@code @ComponentScan("org.my.pkg")} instead of
	 * {@code @ComponentScan(basePackages="org.my.pkg")}.
	 * @return the array of 'basePackages'.
	 */
	String[] value() default {};

	/**
	 * Base packages to scan for annotated components.
	 * <p>
	 * {@link #value()} is an alias for (and mutually exclusive with) this attribute.
	 * <p>
	 * Use {@link #basePackageClasses()} for a type-safe alternative to String-based
	 * package names.
	 * @return the array of 'basePackages'.
	 */
	String[] basePackages() default {};

	/**
	 * Type-safe alternative to {@link #basePackages()} for specifying the packages to
	 * scan for annotated components. The package of each class specified will be scanned.
	 * <p>
	 * Consider creating a special no-op marker class or interface in each package that
	 * serves no purpose other than being referenced by this attribute.
	 * @return the array of 'basePackageClasses'.
	 */
	Class<?>[] basePackageClasses() default {};

	/**
	 * A custom <code>@Configuration</code> for all feign clients. Can contain override
	 * <code>@Bean</code> definition for the pieces that make up the client, for instance
	 * {@link feign.codec.Decoder}, {@link feign.codec.Encoder}, {@link feign.Contract}.
	 *
	 * @see FeignClientsConfiguration for the defaults
	 * @return list of default configurations
	 */
	Class<?>[] defaultConfiguration() default {};

	/**
	 * List of classes annotated with @FeignClient. If not empty, disables classpath
	 * scanning.
	 * @return list of FeignClient classes
	 */
	Class<?>[] clients() default {};

}
```

一般来说，入口及时这个注解，注解本身只是一些配置，比如包路径或者指定具体的class等等

**需要关注的是@Import(FeignClientsRegistrar.class)这一行，@Import注解用于手动导入一些配置或者主键，是spring boot提供的一种方式，通过这种方式可以将主导权交给开发者**

### FeignClientsRegistrar

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {

	private ResourceLoader resourceLoader;

	private Environment environment;

	FeignClientsRegistrar() {
	}

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}

	
}

```

该类实现了ImportBeanDefinitionRegistrar接口，这里看下registerBeanDefinitions方法，做两两件事registerDefaultConfiguration和registerFeignClients

先看registerDefaultConfiguration方法

```java
private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName(), true);

    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        }
        else {
            name = "default." + metadata.getClassName();
        }
        registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
    }
}
```

这里主要是扫描EnableFeignClients的defaultConfiguration属性，并将其注册为bean

接下来看registerFeignClients方法

```java

public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
    Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
    final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);
        scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
        Set<String> basePackages = getBasePackages(metadata);
        for (String basePackage : basePackages) {
            candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
        }
    }
    else {
        for (Class<?> clazz : clients) {
            candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
        }
    }

    for (BeanDefinition candidateComponent : candidateComponents) {
        if (candidateComponent instanceof AnnotatedBeanDefinition) {
            // verify annotated class is an interface
            AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            Assert.isTrue(annotationMetadata.isInterface(), "@FeignClient can only be specified on an interface");

            Map<String, Object> attributes = annotationMetadata
                    .getAnnotationAttributes(FeignClient.class.getCanonicalName());

            String name = getClientName(attributes);
            registerClientConfiguration(registry, name, attributes.get("configuration"));

            registerFeignClient(registry, annotationMetadata, attributes);
        }
    }
}
```

方法中的第一个if-else块是在找需要注册成bean的feign client class，并将其添加到待注册列表中；优先通过@EnableFeignClients注解的clients属性，如果属性为空，则通过value、basePackages中指定的根路径以及basePackageClasses中指定的类所在的路径作为根路径去扫描根路径下标记了@FeignClient注解的类

接下来就是根据FeignClient属性注册bean，重点看registerFeignClient方法

```java
private void registerFeignClient(BeanDefinitionRegistry registry, AnnotationMetadata annotationMetadata,
        Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    Class clazz = ClassUtils.resolveClassName(className, null);
    ConfigurableBeanFactory beanFactory = registry instanceof ConfigurableBeanFactory
            ? (ConfigurableBeanFactory) registry : null;
    String contextId = getContextId(beanFactory, attributes);
    String name = getName(attributes);
    FeignClientFactoryBean factoryBean = new FeignClientFactoryBean();
    factoryBean.setBeanFactory(beanFactory);
    factoryBean.setName(name);
    factoryBean.setContextId(contextId);
    factoryBean.setType(clazz);
    factoryBean.setRefreshableClient(isClientRefreshEnabled());
    BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(clazz, () -> {
        factoryBean.setUrl(getUrl(beanFactory, attributes));
        factoryBean.setPath(getPath(beanFactory, attributes));
        factoryBean.setDecode404(Boolean.parseBoolean(String.valueOf(attributes.get("decode404"))));
        Object fallback = attributes.get("fallback");
        if (fallback != null) {
            factoryBean.setFallback(fallback instanceof Class ? (Class<?>) fallback
                    : ClassUtils.resolveClassName(fallback.toString(), null));
        }
        Object fallbackFactory = attributes.get("fallbackFactory");
        if (fallbackFactory != null) {
            factoryBean.setFallbackFactory(fallbackFactory instanceof Class ? (Class<?>) fallbackFactory
                    : ClassUtils.resolveClassName(fallbackFactory.toString(), null));
        }
        return factoryBean.getObject();
    });
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    definition.setLazyInit(true);
    validate(attributes);

    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);
    beanDefinition.setAttribute("feignClientsRegistrarFactoryBean", factoryBean);

    // has a default, won't be null
    boolean primary = (Boolean) attributes.get("primary");

    beanDefinition.setPrimary(primary);

    String[] qualifiers = getQualifiers(attributes);
    if (ObjectUtils.isEmpty(qualifiers)) {
        qualifiers = new String[] { contextId + "FeignClient" };
    }

    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, qualifiers);
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);

    registerOptionsBeanDefinition(registry, contextId);
}

```

上面方法就是构造了一个BeanDefinition并注册，BeanDefinition是bean的元数据描述，根据代码，我们可以得知beanFactory是FeignClientFactoryBean，同时在构建BeanDefinition还做了一些准备工作，例如对url和path的处理，这里不具体分析。生成bean式通过BeanFactor也就是FeignClientFactoryBean来实现的

接下来我们看FeignClientFactoryBean

```java
public class FeignClientFactoryBean
		implements FactoryBean<Object>, InitializingBean, ApplicationContextAware, BeanFactoryAware {

	@Override
	public Object getObject() {
		return getTarget();
	}

	@Override
	public Class<?> getObjectType() {
		return type;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}

	public Class<?> getType() {
		return type;
	}

}

```

这里getObjectType返回的是bean的类型，其实就是标记了@FeignClient的接口类型，isSingleton方法可以得知bean都是单例的，getObject返回的就是具体的bean，这里是调用了getTarget方法来返回的，接下来看getTarget方法

```java
/**
 * @param <T> the target type of the Feign client
 * @return a {@link Feign} client created with the specified data and the context
 * information
 */
<T> T getTarget() {
    FeignContext context = beanFactory != null ? beanFactory.getBean(FeignContext.class)
            : applicationContext.getBean(FeignContext.class);
    Feign.Builder builder = feign(context);

    if (!StringUtils.hasText(url)) {

        if (LOG.isInfoEnabled()) {
            LOG.info("For '" + name + "' URL not provided. Will try picking an instance via load-balancing.");
        }
        if (!name.startsWith("http")) {
            url = "http://" + name;
        }
        else {
            url = name;
        }
        url += cleanPath();
        return (T) loadBalance(builder, context, new HardCodedTarget<>(type, name, url));
    }
    if (StringUtils.hasText(url) && !url.startsWith("http")) {
        url = "http://" + url;
    }
    String url = this.url + cleanPath();
    Client client = getOptional(context, Client.class);
    if (client != null) {
        if (client instanceof FeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
        }
        if (client instanceof RetryableFeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((RetryableFeignBlockingLoadBalancerClient) client).getDelegate();
        }
        builder.client(client);
    }
    Targeter targeter = get(context, Targeter.class);
    return (T) targeter.target(this, builder, context, new HardCodedTarget<>(type, name, url));
}
```

这里最要看最后两行，通常targeter返回的是DefaultTargeter，具体可以查看FeignAutoConfiguration

接下来查看DefaultTargeter.target方法

```java
class DefaultTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
			Target.HardCodedTarget<T> target) {
		return feign.target(target);
	}

}
```

实际上是Feign.Builder.target方法

```javascript
public <T> T target(Target<T> target) {
  return build().newInstance(target);
}
```

再看build方法

```java
public Feign build() {
      Client client = Capability.enrich(this.client, capabilities);
      Retryer retryer = Capability.enrich(this.retryer, capabilities);
      List<RequestInterceptor> requestInterceptors = this.requestInterceptors.stream()
          .map(ri -> Capability.enrich(ri, capabilities))
          .collect(Collectors.toList());
      Logger logger = Capability.enrich(this.logger, capabilities);
      Contract contract = Capability.enrich(this.contract, capabilities);
      Options options = Capability.enrich(this.options, capabilities);
      Encoder encoder = Capability.enrich(this.encoder, capabilities);
      Decoder decoder = Capability.enrich(this.decoder, capabilities);
      InvocationHandlerFactory invocationHandlerFactory =
          Capability.enrich(this.invocationHandlerFactory, capabilities);
      QueryMapEncoder queryMapEncoder = Capability.enrich(this.queryMapEncoder, capabilities);

      SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
          new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
              logLevel, decode404, closeAfterDecode, propagationPolicy, forceDecoding);
      ParseHandlersByName handlersByName =
          new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
              errorDecoder, synchronousMethodHandlerFactory);
      return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

实际上返回了ReflectiveFeign

再看ReflectiveFeign.netInstance方法

```java
 public <T> T newInstance(Target<T> target) {
    // ...
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```

T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class\<?>\[] {target.type()}, handler)可以看到，这里返回的是一个jdk动态代理对象，需要重点关注的是handler，往上回溯，可以看到handler，是通过factory.create(target, methodToHandler)创建，具体方式是InvocationHandlerFactory.create

```java
public interface InvocationHandlerFactory {

  InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);

  /**
   * Like {@link InvocationHandler#invoke(Object, java.lang.reflect.Method, Object[])}, except for a
   * single method.
   */
  interface MethodHandler {
    Object invoke(Object[] argv) throws Throwable;
  }

  static final class Default implements InvocationHandlerFactory {

    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
  }
}
```

实际上返回的是FeignInvocationHandler，实现了InvocationHandler

```java
static class FeignInvocationHandler implements InvocationHandler {

private final Target target;
private final Map<Method, MethodHandler> dispatch;

FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
  this.target = checkNotNull(target, "target");
  this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
}

@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  if ("equals".equals(method.getName())) {
    try {
      Object otherHandler =
          args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
      return equals(otherHandler);
    } catch (IllegalArgumentException e) {
      return false;
    }
  } else if ("hashCode".equals(method.getName())) {
    return hashCode();
  } else if ("toString".equals(method.getName())) {
    return toString();
  }

  return dispatch.get(method).invoke(args);
}
}
```

看下invoke方法，可以看到，实际上跟从dispatch中找到具体method对应的实现方法调用，dispatch需要回溯到ReflectiveFeign.netInstance方法

```java
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    // ...
}
```

这里实际是通过反射，然后为每个方法的创建了一个MethodHandler。最终起作用的是methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));

nameToHandler = targetToHandlersByName.apply(target);

```java
public Map<String, MethodHandler> apply(Target target) {
  List<MethodMetadata> metadata = contract.parseAndValidateMetadata(target.type());
  Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
  for (MethodMetadata md : metadata) {
    BuildTemplateByResolvingArgs buildTemplate;
    if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
      buildTemplate =
          new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
    } else if (md.bodyIndex() != null) {
      buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
    } else {
      buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder, target);
    }
    if (md.isIgnored()) {
      result.put(md.configKey(), args -> {
        throw new IllegalStateException(md.configKey() + " is not a method handled by feign");
      });
    } else {
      result.put(md.configKey(),
          factory.create(target, md, buildTemplate, options, decoder, errorDecoder));
    }
  }
  return result;
}

```

for循环中，上面的if-else块实际上在解析方法上的一些参数，比如@RequestParam,@RequestBody等然后根据不同参数构造不同的BuildTemplateByResolvingArgs，最终通过factory.create返回一个MethodHandler

具体SynchronousMethodHandler.Factory.create方法

```java
 public MethodHandler create(Target<?> target,
                                MethodMetadata md,
                                RequestTemplate.Factory buildTemplateFromArgs,
                                Options options,
                                Decoder decoder,
                                ErrorDecoder errorDecoder) {
      return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger,
          logLevel, md, buildTemplateFromArgs, options, decoder,
          errorDecoder, decode404, closeAfterDecode, propagationPolicy, forceDecoding);
}
```

实际返回了一个SynchronousMethodHandler

```java
final class SynchronousMethodHandler implements MethodHandler {

  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
 
}

```

实际上就是构造一个RequestTemplate，然后通过excuteAndDecode发送请求并解析成具体的返回类型RequestTemplate template = buildTemplateFromArgs.create(argv);就不在详细跟

主要看SynchronousMethodHandler.executeAndDecode方法

```java
  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 12
      response = response.toBuilder()
          .request(request)
          .requestTemplate(template)
          .build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);


    if (decoder != null)
      return decoder.decode(response, metadata.returnType());

    CompletableFuture<Object> resultFuture = new CompletableFuture<>();
    asyncResponseHandler.handleResponse(resultFuture, metadata.configKey(), response,
        metadata.returnType(),
        elapsedTime);

    try {
      if (!resultFuture.isDone())
        throw new IllegalStateException("Response handling not done");

      return resultFuture.join();
    } catch (CompletionException e) {
      Throwable cause = e.getCause();
      if (cause != null)
        throw cause;
      throw e;
    }
  }

```

Request request = targetRequest(template); 需要关注下，这里可以对RequestTemplate做增强

```java
 Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    return target.apply(template);
  }
```

也就是说我们可以自定义RequestInterceptor来做一些自定义的增强

response = client.execute(request, options);发送http请求

然后剩下的部分就是重试和返回结果解析

至此，源码分析结束

