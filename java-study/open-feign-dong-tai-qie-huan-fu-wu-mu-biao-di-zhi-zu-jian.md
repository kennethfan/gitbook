# Open feign动态切换服务目标地址组件

在微服务架构中，我们经常需要调用其他服务的API。随着业务的发展，可能会出现需要在多个服务集群之间动态切换的场景，例如灰度发布、故障转移、负载均衡等。本文介绍一个基于 Spring Boot 和 OpenFeign 的动态切换服务提供方集群的组件，它允许在运行时根据配置动态切换 Feign 调用的下游集群，而无需重启服务。

## 组件背景

在复杂的微服务环境中，为了提高系统的可用性和稳定性，通常会部署多个服务实例集群（如生产/灰度、蓝绿）。传统通过修改配置或代码并重启服务的方式不够灵活。为了解决这个问题，开发了一个动态切换服务提供方集群的 Spring Boot Starter 组件。该组件基于注解与配置，可在运行时动态切换 Feign 调用的目标集群。

## 核心设计思路（步骤化概览）

#### 路由键注解 - @RoutingKey

用于标记需要动态路由的方法或类。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface RoutingKey {
    String value() default "";
}
```

@RoutingKey 注解可以标记在方法或类上，用于指定路由规则。支持 SpEL 表达式，可以根据方法参数动态确定路由键。

### RoutingKeyInterceptor

拦截被 @RoutingKey 注解标记的方法，根据注解值和配置确定目标 URL 并放入上下文。

```java
@Slf4j
public class RoutingKeyInterceptor implements MethodInterceptor {

  private AnnotationClassResolver resolver;
  private Environment environment;
  private final DefaultParameterNameDiscoverer discoverer = new DefaultParameterNameDiscoverer();

  public RoutingKeyInterceptor(AnnotationClassResolver resolver, Environment environment) {
    this.resolver = resolver;
    this.environment = environment;
  }

  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    String url = determineUrl(invocation);
    try {
      RequestUrlContext.set(url);
      return invocation.proceed();
    } finally {
      RequestUrlContext.clear();
    }
  }

  private String determineUrl(MethodInvocation invocation) {
    String expr = resolver.getValue(invocation.getMethod(), invocation.getThis().getClass());
    if (StrUtil.isBlank(expr)) {
      if (log.isDebugEnabled()) {
        log.debug("RoutingKeyInterceptor.determineUrl, no expr found, method={}", invocation.getMethod());
      }
      return StrUtil.EMPTY;
    }

    String routingKey = calculateExpr(expr, invocation.getMethod(), invocation.getArguments());
    if (StrUtil.isBlank(routingKey)) {
      if (log.isDebugEnabled()) {
        log.debug("RoutingKeyInterceptor.determineUrl, expr value blank, expr={}, method={}, args={}", expr, invocation.getMethod(), Arrays.asList(invocation.getArguments()));
      }
      return StrUtil.EMPTY;
    }

    String url = environment.getProperty(routingKey, StrUtil.EMPTY);
    if (log.isDebugEnabled()) {
      log.debug("RoutingKeyInterceptor.determineUrl, end, url={}, routingKey={}", url, routingKey);
    }

    return url;
  }

  private String calculateExpr(String expr, Method method, Object[] params) {
    String exprValue = "unknown";
    if (StrUtil.isNotBlank(expr) && expr.contains("#")) {
      ExpressionParser parser = new SpelExpressionParser();
      StandardEvaluationContext ctx = new StandardEvaluationContext();
      String[] parameterNames = discoverer.getParameterNames(method);
      if (parameterNames != null) {
        for (int i = 0; i < parameterNames.length; i++) {
          ctx.setVariable(parameterNames[i], params[i]);
        }
      }
      Expression expression = parser.parseExpression(expr);
      Object value = expression.getValue(ctx);
      if (value != null) {
        exprValue = value.toString();
      }
    }

    return exprValue;
  }
}
```

RoutingKeyInterceptor 负责解析 @RoutingKey 注解中的表达式，根据方法参数计算出路由键，从环境中取出对应的 URL，并将其存入 RequestUrlContext。

### DynamicClusterRequestInterceptor（Feign 请求拦截器）

在 Feign 请求发送前修改目标 URL。

```java
@Slf4j
public class DynamicClusterRequestInterceptor implements RequestInterceptor {

  private Collection<Class<?>> supportTypes;
  private RequestRouter requestRouter;

  public DynamicClusterRequestInterceptor(Collection<Class<?>> supportTypes, RequestRouter requestRouter) {
    this.supportTypes = supportTypes;
    this.requestRouter = requestRouter;
  }

  @Override
  public void apply(RequestTemplate template) {
    Class<?> type = template.feignTarget().type();
    if (!typeMatch(type)) {
      if (log.isDebugEnabled()) {
        log.debug("DynamicClusterRequestInterceptor.apply, not match, type={}", type.getName());
      }
      return;
    }

    String url = requestRouter.determineUrl(template);
    if (StrUtil.isBlank(url)) {
      if (log.isDebugEnabled()) {
        log.debug("DynamicClusterRequestInterceptor.apply, route url is blank, type={}", type.getName());
      }
      return;
    }

    template.target(url);
  }

  private boolean typeMatch(Class<?> type) {
    for (Class<?> supportType : supportTypes) {
      if (type.isAssignableFrom(supportType)) {
        return true;
      }
    }

    return false;
  }
}
```

该拦截器通过 RequestRouter 获取目标 URL，并调用 template.target(url) 修改请求目标。

### RequestUrlContext（请求上下文）

用于在线程/调用链间传递目标 URL（示例中使用 SkyWalking 的 TraceContext）。

```java
public class RequestUrlContext {

  public static final String KEY = "routing-url";

  public static void set(String url) {
    TraceContext.putCorrelation(KEY, url);
  }

  public static String url() {
    return TraceContext.getCorrelation(KEY).orElse(null);
  }

  public static void clear() {
    TraceContext.putCorrelation(KEY, StrUtil.EMPTY);
  }
}
```

### ContextRequestRouter（上下文路由器）

从 RequestUrlContext 中获取目标 URL，作为 RequestRouter 的实现之一。

```java
public class ContextRequestRouter implements RequestRouter {

  @Override
  public String determineUrl(RequestTemplate requestTemplate) {
    return RequestUrlContext.url();
  }
}
```

### 启用配置 - @EnabledRoutingKeyAdvisor / DynamicClusterConfiguration

启用路由键顾问功能并注册 AOP advisor：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DynamicClusterConfiguration.class})
public @interface EnabledRoutingKeyAdvisor {
}
```

DynamicClusterConfiguration 示例：

```java
@Configuration
public class DynamicClusterConfiguration {

  @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
  @Bean
  public Advisor routingKeyAnnotationAdvisor(Environment environment) {
    RoutingKeyInterceptor interceptor = new RoutingKeyInterceptor(new AnnotationClassResolver(true, RoutingKey.class), environment);
    AnnotationAdvisor advisor = new AnnotationAdvisor(interceptor, RoutingKey.class);
    return advisor;
  }
}
```

DynamicClusterConfiguration 创建了一个 AOP 顾问，用于拦截被 @RoutingKey 注解标记的方法。

## 核心代码解析（按模块复列）

* @RoutingKey 注解：标记方法或类，支持 SpEL 表达式。
* RoutingKeyInterceptor：解析注解表达式，计算路由键，从 Environment 获取 URL，放入 RequestUrlContext。
* RequestUrlContext：使用 TraceContext 存储目标 URL（便于分布式追踪）。
* DynamicClusterRequestInterceptor：Feign 请求拦截器，根据 RequestRouter 返回的 URL 修改请求目标。
* ContextRequestRouter：从 RequestUrlContext 获取 URL。

## 使用示例（步骤化）

### 启用功能（在 Spring Boot 主类上添加注解）

```java
@SpringBootApplication
@EnableRoutingKeyAdvisor
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

（注：本文前面定义的注解为 @EnabledRoutingKeyAdvisor，示例中使用了 @EnableRoutingKeyAdvisor，请根据实际使用的注解名保持一致。）

### 在 Feign 客户端方法上使用 @RoutingKey

```java
@FeignClient(name = "user-service")
public interface UserServiceClient {
    
    @GetMapping("/users/{id}")
    @RoutingKey("user.service.url")
    User getUserById(@PathVariable("id") Long id);
    
    @PostMapping("/users")
    @RoutingKey("#user.type + '.user.service.url'")
    User createUser(@RequestBody User user);
}
```

示例展示了静态路由键（"user.service.url"）和基于方法参数的 SpEL 表达式路由键（"#user.type + '.user.service.url'"）。

### 在配置文件中定义路由键对应的 URL

示例（application.yml）：

```yaml
user:
  service:
    url: http://user-service:8080
vip:
  user:
    service:
      url: http://vip-user-service:8080
```

通过修改这些配置（例如由灰度服务的 URL 切换到生产服务的 URL），配合运行时更改 Environment 或其它配置源的话，可以在不重启的情况下改变路由目标。

## 总结

该 Spring Boot Starter 组件提供了一种灵活、无侵入的方式，在运行时动态切换 Feign 调用的下游集群，适用于灰度发布、故障切换、蓝绿部署等场景。主要优势包括：

* 无侵入性：通过注解使用，对业务代码侵入小；
* 动态性：支持运行时动态切换，无需重启服务；
* 灵活性：支持 SpEL 表达式，可根据方法参数动态确定路由目标；
* 可追踪性：示例中集成 SkyWalking TraceContext，支持分布式追踪。

通过上述组件组合（注解 + AOP 拦截 + Feign 请求拦截器 + 请求上下文），可以在复杂微服务场景中更好地管理服务调用，提高系统灵活性与稳定性。
