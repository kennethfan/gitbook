---
description: springmvc 动态注册路由
---

# springmvc 动态注册路由

## 引言

对于javaer来说，做web开发时springmvc是经常使用的框架。使用springmvc可以很方便的添加一个路由，只需要使用@Controller注解或者@RestController注解即可。在做业务开发时我们几乎都是这么做的，但是当我们开发的是通用组件时，如果使用这种方式，就会侵入业务代码，或者需要业务多做一些额外的配置。因此，我们需要有其他方式来做动态路由的注册

## 目标

业务方不需要添加Controller或者包含@Controller注解的bean即可完成路由注册

## 方案

在springmvc框架中，实际上提供了类似的组件，这个组件就是RequestMappingHandlerMapping，全路径org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

```java
@Override
public void registerMapping(RequestMappingInfo mapping, Object handler, Method method) {
    super.registerMapping(mapping, handler, method);
    updateConsumesCondition(mapping, method);
}
```

通过registerMapping即可，参数如下

mapping(RequestMappingInfo)，标识请求的path信息，可以通过内置的方法快速构造

```java
 RequestMappingInfo.paths("/foo", "/bar").build()
```

hanlder(Object)，method(Method)，这两个是一对，分别表示处理请求的方法所属的类实例和具体的方法

RequestMappingHandlerMapping可以直接注入，在springmvc启动时会自动创建

&#x20;动态注册路由如下

<pre class="language-java"><code class="lang-java">requestMappingHandlerMapping.registerMapping(
<strong>    RequestMappingInfo.paths("/foo").build(),
</strong>    executorEndpoint,
    ReflectUtil.getMethod(ExecutorEndpoint.class, "foo", String.class)
);
</code></pre>

至此，动态路由注册完成

