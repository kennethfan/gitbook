---
description: spring cloud应用分布式链路跟踪简易实现
---

# spring cloud应用分布式链路跟踪简易实现

## 背景

在一个项目中，一个请求通常会有调用各种上下游，最常见的有数据库、缓存、消息队列以及下游服务；在链路比较复杂的时，定位问题以及分析问题会非常麻烦，因此需要通过某种方式，定位一次请求的所有上下游链路

## 目标

实现单次请求的上下游链路跟踪

## 现状

1、服务基于spring cloud，网关使用spring cloud gateway

2、服务间调用使用open feign调用，没有rpc

3、使用mybatis-plus操作数据库

4、使用redis缓存

5、暂时没有使用消息队列

## 业界方案

spring cloud 通常spring slueth + zipkin/skywalking实现

1、slueth实现链路traceId + spanId生成

2、数据上报zipkin/skywalking实现数据展示

我们当前的主要问题在于，预算不够，暂时无法申请资源部署zipkin/skywalking；因此可以考虑只使用slueth实现traceId + spanId生成，待需要时再接入zipkin/skywalking或者其他日志收集系统（比如ELK，SLS等）

这里更多的是探讨一种简易实现，只生成traceId + spanId，结合本地日志系统，实现方案定位问题即可

## 简易方案

实现的重点在于如何生成traceId和spanId，并打印到日志系统

traceId：标识一次请求；因此同一请求的上下游链路，traceId必须相同，不同请求traceId必须不相同

spanId：标识一次请求中的不同调用链路，需要可以通过spanId还原出链路调用时序

### traceId生成

这里直接使用uuid

### spanId

我们通过x.y.z格式表示

相同长度的spanId标识双方处于同一层次；比如0.1和0.2表示一次请求的第1次和第2次子链路

相同前缀，长度较长者为子调用；比如0.1.1表示0.1链路的子链路

整体格式如下

```
0.1 start 
   ...
   0.1.1 start
   ...
   0.1.1 end
   ...
   0.1.2 start
     0.1.2.1 start
     0.1.2.1 end
     ...
   0.1.2 end
0.1 end
   
0.2 start
  ...
0.2 end

...
```

spanId如何生成接下来会细说

### 入口

入口主要3个

1. 网关(http或者rpc)
2. 各类定时任务
3. 消息队列消费者（本项目暂时不涉及）

### 传递

主要几种

1. http header头
2. rpc header或者扩展字段（本项目不涉及）
3. 消息队列传递

### 实现

#### traceId、spanId生成

```java
public class TraceContext {

    public static final String ROOT_SPAN_ID = "0";

    /**
     * 上下文，主要记录当前层级的spanId
     */
    private static final ThreadLocal<SpanGenerator> SPAN_GENERATOR = new ThreadLocal<>();

    private static class SpanGenerator {
        /**
         * 当前层级spanId
         */
        private String parentId;

        private AtomicInteger counter = new AtomicInteger();

        private SpanGenerator(String parentId) {
            this.parentId = parentId;
        }

        /**
         * 子链路spanId，当前层级spanId+计数器
         * @return
         */
        public String nextSpanId() {
            return parentId + counter.incrementAndGet();
        }
    }

    public static void setParentId(String parentId) {
        SPAN_GENERATOR.set(new SpanGenerator(StringUtils.defaultIfEmpty(parentId, ROOT_SPAN_ID) + "."));
    }

    /**
     * traceId生成
     * @return
     */
    public static String traceId() {
        return UUID.randomUUID().toString().replaceAll("-", "");
    }

    /**
     * spanId生成
     * @return
     */
    public static String nextSpanId() {
        SpanGenerator spanGenerator = SPAN_GENERATOR.get();
        if (spanGenerator == null) {
            return null;
        }

        return spanGenerator.nextSpanId();
    }
}
```

#### 入口传递

**网关**

```java
public class GatewayTraceFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 网关作为流量入口，因此traceId需要重新生成，spanId使用根spanId即可
        final String traceId = TraceContext.traceId();
        exchange.getRequest().mutate().headers(httpHeaders -> {
            httpHeaders.add(TraceStateEnum.TraceIdName.getValue(), traceId);
            httpHeaders.add(TraceStateEnum.SpanIdName.getValue(), TraceContext.ROOT_SPAN_ID);
        });

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // 将traceId放入response header，方便定位问题
            HttpHeaders headers = exchange.getResponse().getHeaders();
            headers.add("X-API-Trace", traceId);
        }));
    }

    @Override
    public int getOrder() {
        // 过滤器的执行顺序，可以根据需要设置
        return 2;
    }
}
```

**定时任务**

定时任务主要使用spring task通过@Scheduled定义任务

通过@EnableScheduling注解开始代码跟踪，发现并没有提供可以针对task扩展的部分；因此此处使用AOP实现

```java
@Slf4j
@Aspect
@Component
public class ScheduleAspect {
    @Pointcut("@annotation(org.springframework.scheduling.annotation.Scheduled)")
    public void pointcut() {
    }

    @Around(value = "pointcut()")
    public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
        log.debug("ScheduleAspect.doAround call");
        try {
            MDC.put(TraceStateEnum.TraceIdName.getValue(), TraceContext.traceId());
            MDC.put(TraceStateEnum.SpanIdName.getValue(), TraceContext.ROOT_SPAN_ID);
            return pjp.proceed();
        } finally {
            MDC.clear();
        }
    }
}
```

#### 传递

http调用通过open feign，因此针对feign做拦截，在调用时将traceId和spanId放入header即可

```java
public class TraceIdFeignInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 获取MDC中的tid
        String traceId = MDC.get(TraceStateEnum.TraceIdName.getValue());

        // 设置请求头
        template.header(TraceStateEnum.TraceIdName.getValue(), traceId);
        // 这里放的是新生成的子链路spanId
        template.header(TraceStateEnum.SpanIdName.getValue(), TraceContext.nextSpanId());
    }
}

@Configuration
public class FeignConfiguration {
    
    @Bean
    public RequestInterceptor traceIdFeignInterceptor() {
        return new TraceIdFeignInterceptor();
    }
}
```

针对数据库、缓存的传递，自定义实现暂时也没有什么好的方法，要么通过AOP，要么通过Java agent做增强，或者手动埋点（代码侵入性较强，一般不这么做）；这里暂时先忽略不做

针对消息队列，通常可以放在Message扩展字段里面传递，这里不做探讨

#### 接收

http接收比较简单，通过springmvc interceptor即可

```java
@Component
@Slf4j
public class TraceIdInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 从请求头中获取 Trace ID
        String traceId = request.getHeader(TraceStateEnum.TraceIdName.getValue());
        String spanId = request.getHeader(TraceStateEnum.SpanIdName.getValue());

        // 将 Trace ID 添加到 MDC
        MDC.put(TraceStateEnum.TraceIdName.getValue(), traceId);
        MDC.put(TraceStateEnum.SpanIdName.getValue(), spanId);
        TraceContext.setParentId(spanId);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 在请求处理完成后，清理 MDC 值
        try {
            MDC.clear();
        } catch (IllegalArgumentException e) {
            log.error("MDC.remove error", e.getMessage());
        }
    }
}
```

消息队列同理，在Message扩展字段中去取即可

#### 异步执行处理

**使用自定义线程池**

通常两种方式，一种是提交任务时，使用封装过的callable或者runnable

代码如下

```java
public class TraceRunnable implements Runnable {
    private Runnable runnable;

    public TraceRunnable(Runnable runnable) {
        this.runnable = runnable;
    }

    @Override
    public void run() {
        try {
            final Map<String, String> contextMap = MDC.getCopyOfContextMap();
            if (contextMap != null) {
                MDC.setContextMap(contextMap);
            }
            runnable.run();
        } finally {
            MDC.clear();
        }
    }
}
```

callable同理

还有一种是针对线程池做封装

```java
public class TraceExecutorService implements ExecutorService {
    private ExecutorService delegate;

    @NotNull
    @Override
    public <T> Future<T> submit(@NotNull Callable<T> task) {
        return this.delegate.submit(new TraceCallable<>(task));
    }
    
    // ... 其他方法类似
}
```

**@Async注解**

一种是使用了自定义线程池，参考上面

还有一种使用默认线程池，通过@EnableAsync注解个跟踪，可以通过实现自定义的AsyncConfigurer实现

默认是AsyncConfigurerSupport

```java
@Slf4j
@Component
public class TraceAsyncConfigurer extends AsyncConfigurerSupport {

    @Autowired
    private TaskExecutorBuilder taskExecutorBuilder;

    @Override
    public Executor getAsyncExecutor() {
        log.info("TraceAsyncConfigurer.getAsyncExecutor call");
        return taskExecutorBuilder.build();
    }
}
```

TaskExecutorBuilder是spring系统内部实现，可以自动感知到对人物的增强

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

所以只需要自定一个TaskDecorator即可

```java
@Slf4j
@Component
public class TraceTaskDecorate implements TaskDecorator, InitializingBean {

    public void afterPropertiesSet() throws Exception {
        log.info("TraceTaskDecorate init");
    }

    public Runnable decorate(Runnable runnable) {
        final Map<String, String> contextMap = MDC.getCopyOfContextMap();
        if (contextMap == null) {
            return runnable;
        }

        return () -> {
            try {
                MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

#### 日志配置

```markup
<appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${log.path}/log_warn.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [TraceId:%X{traceId}] [SpanId:%X{spanId}] [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        ...
</appender>
```
