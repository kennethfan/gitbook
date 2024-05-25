---
description: Feign调用转成本地方法调用
---

# Feign调用转成本地方法调用

## 背景

最近项目因为某些原因，将服务A和服务B合并成了一个服务（参考上篇：基于spring cloud gateway & springmvc服务合并）。合并完成之后原来服务A和服务B通信，通过Feign调用由原来的链路服务A->服务B变成了服务A->服务A；合并完的调用链路一次外部请求会消耗更多的web容器资源，间接降低服务A的对外提供服务能力

## 目标

优化调用链路，降低无谓web容器资源消耗

## 思路

既然服务B已经合并到服务A了，实际上原有的服务B的http接口直接通过本地方法去调用就可以。

但是，如果一个个调用处去直接改，工作量极大；且后期资源足够的情况下服务B再拆出来，又需要再次替换为Feign调用。能否通过强大的AOP，在尽可能少的代码改动下实现呢；答案是可以的

## 实施

### 找到接口和本地方法的映射

我们知道，springmvc每个url都会对应一个Controller的方法(HandlerMethod)，所以我们只需要将服务B的所有controller都扫描一遍，即可建立对应的映射关系

那如何拿到这些controller呢？

1. 拿到ServletContext
2. 通过ServletContext拿到WebApplicationContext
3. 通过WebApplicationContext拿到所有的RequestMappingHandlerMapping
4. 通过RequestMappingHandlerMapping拿到HandlerMethod
5. 解析HandlerMethod拿到url和对应的RequestMethods

参考代码如下

```java
private Map<String /** url **/, Map.Entry<HandlerMethod, Set<RequestMethod>>> parseServiceBHandlerMethodMapping() {
    if (servletContext == null) {
        return new HashMap<>();
    }

    final Map<String /** url **/, Map.Entry<HandlerMethod, Set<RequestMethod>>> handlerMethodMapping = new HashMap<>();

    WebApplicationContext webContext = WebApplicationContextUtils.getWebApplicationContext(servletContext);
    Map<String, HandlerMapping> allRequestMappings = BeanFactoryUtils.beansOfTypeIncludingAncestors(webContext,
            HandlerMapping.class, true, false);
    if (allRequestMappings.size() == 0) {
        return handlerMethodMapping;
    }

    for (HandlerMapping handlerMapping : allRequestMappings.values()) {
        if (!(handlerMapping instanceof RequestMappingHandlerMapping)) {
            continue;
        }

        RequestMappingHandlerMapping requestMappingHandlerMapping = (RequestMappingHandlerMapping) handlerMapping;
        Map<RequestMappingInfo, HandlerMethod> handlerMethods = requestMappingHandlerMapping.getHandlerMethods();
        for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : handlerMethods.entrySet()) {
            HandlerMethod handlerMethod = entry.getValue();
            // 这里只处理ServiceB 包路径下的controller
            if (!handlerMethod.getMethod().getDeclaringClass().getName().contains(".serviceB.")) {
                continue;
            }

            // 这里只针对加上了@LocalCall注解的方法去解析，有些方法依赖http上下文，不能直接转为本地方法调用
            if (handlerMethod.getMethodAnnotation(LocalCall.class) == null) {
                continue;
            }

            RequestMappingInfo requestMappingInfo = entry.getKey();
            if (CollectionUtils.isEmpty(requestMappingInfo.getDirectPaths())) {
                continue;
            }

            Set<RequestMethod> requestMethodSet = requestMappingInfo.getMethodsCondition().getMethods();
            if (CollectionUtils.isEmpty(requestMethodSet)) {
                continue;
            }

            log.info("parseServiceBHandlerMethodMapping, urls={}, request_methods={}, method={}",
                    requestMappingInfo.getDirectPaths(),
                    requestMappingInfo.getMethodsCondition().getMethods(),
                    handlerMethod.getMethod());
            for (String url : requestMappingInfo.getDirectPaths()) {
                // URL_PREFIX实际是ServiceA的context-path
                handlerMethodMapping.put(URL_PREFIX + url, new AbstractMap.SimpleImmutableEntry(handlerMethod, requestMethodSet));
            }
        }
    }

    return handlerMethodMapping;
}
```

### 解析ServiceB FeignClient所有方法，找到可以转为本地方法的method，并建立映射关系

FeignClient接口方法上通常都会有几个注解@RequestMapping/@GetMapping/@PostMapping

所以只需要解析到这几个注解，就可以拿到对应方法请求的url和RequestMethod

结合上面拿到的url和HandlerMethod的映射关系，我们就可以建立FeignClient中的方法和HandlerMethod的映射关系

代码如下

```java
public void onApplicationEvent(ApplicationStartedEvent event) {
    Map<String /** url **/, Map.Entry<HandlerMethod, Set<RequestMethod>>> handlerMethodMapping = parseServiceBHandlerMethodMapping();
    if (handlerMethodMapping.size() == 0) {
        return;
    }

    Class<?> clazz;
    try {
        clazz = Class.forName("ServiceBFeignClient class name");
    } catch (ClassNotFoundException e) {
        return;
    }

    Method[] methods = clazz.getMethods();
    if (methods.length == 0) {
        return;
    }

    for (Method method : methods) {
        HandlerMethod handlerMethod = getHandlerMethod(handlerMethodMapping, method);
        if (handlerMethod == null) {
            continue;
        }

        // 缓存下来，后面直接用
        METHOD_MAPPING.put(method, handlerMethod);
    }
}

private HandlerMethod getHandlerMethod(Map<String /** url **/, Map.Entry<HandlerMethod, Set<RequestMethod>>> urlMapping, Method method) {
    Map.Entry<String[] /** url **/, RequestMethod[]> entry = parseMethodInfo(method);
    if (entry == null) {
        return null;
    }

    Map.Entry<HandlerMethod, Set<RequestMethod>> mappingEntry = urlMapping.get(entry.getKey()[0]);
    if (mappingEntry == null) {
        return null;
    }

    // RequestMethod也要可以匹配才行
    if (Arrays.stream(entry.getValue()).anyMatch(o -> mappingEntry.getValue().contains(o))) {
        log.info("getHandlerMethod, method={}, controller_method={}",
                method, mappingEntry.getKey().getMethod());
        return mappingEntry.getKey();
    }

    return null;
}

/**
 * 方法比较简单，就是解析注解拿到url和RequestMethod
 */
private Map.Entry<String[], RequestMethod[]> parseMethodInfo(Method method) {
    RequestMapping requestMapping = method.getAnnotation(RequestMapping.class);
    if (requestMapping != null) {
        String[] urls = requestMapping.value();
        if (urls.length == 0) {
            return null;
        }

        RequestMethod[] requestMethods = requestMapping.method();
        if (requestMethods.length == 0) {
            return null;
        }

        return new AbstractMap.SimpleImmutableEntry<>(urls, requestMethods);
    }

    GetMapping getMapping = method.getAnnotation(GetMapping.class);
    if (getMapping != null) {
        String[] urls = getMapping.value();
        if (urls.length == 0) {
            return null;
        }

        return new AbstractMap.SimpleImmutableEntry<>(urls, new RequestMethod[]{RequestMethod.GET});
    }

    PostMapping postMapping = method.getAnnotation(PostMapping.class);
    if (postMapping != null) {
        String[] urls = postMapping.value();
        if (urls.length == 0) {
            return null;
        }

        return new AbstractMap.SimpleImmutableEntry<>(urls, new RequestMethod[]{RequestMethod.POST});
    }

    return null;
}

```

### 定义AOP

参考代码如下

```java
@Pointcut("execution(* ServiceBFeignClient.*(..))")
public void pointcut() {
}

@Around(value = "pointcut()")
public Object doAround(ProceedingJoinPoint pjp) throws Throwable {
    if (enableLocalCall()) {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        HandlerMethod handlerMethod = METHOD_MAPPING.get(method);
        if (handlerMethod != null) {
            log.info("doAround, method={}, controller_method={}, args={}",
                    method, handlerMethod.getMethod(), pjp.getArgs());
            // 拿到对应的Controller
            Object obj = applicationContext.getBean(handlerMethod.getBeanType());
            // 放射调用
            return handlerMethod.getMethod().invoke(obj, pjp.getArgs());
        }
    }

    return pjp.proceed();
}

private boolean enableLocalCall() {
    // 开关控制AOP是否实时生效
    return propertyHelper.getPropertyBoolean("aop switch key");
}
```
