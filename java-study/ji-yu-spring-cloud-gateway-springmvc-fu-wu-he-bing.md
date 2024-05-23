---
description: 基于Spring Cloud gateway & Springmvc服务合并
---

# 基于Spring Cloud gateway & Springmvc服务合并

## 背景

当前项目中，服务拆分的粒度太小，每个服务都需要单独启动一个jvm进程，导致服务器内存不足，因此需要对微服务做合并

## 目标

合并服务，降低内存使用

## 现状

服务A：业务服务，和具体业务逻辑相关；url = /serviceA/\*\*, context-path=/serviceA

服务B：通用服务，主要是发送短信，文件上传等功能；url = /serviceB/\*\*, context-path=/serviceB

1、服务A和服务B的subpath没有冲突

2、服务A和服务B的数据库是同一个

3、application.yml只有基础的内容，大部分配置都在nacos上

因此，可以考虑直接合并

具体思路是，服务B变成一个通用java jar，由服务A引入

## 问题

1、原有的路由规则需要调整

2、serviceB的url将会从/serviceB/\*\*变成/serviceB/\*\*

## 实施

### serviceB变成普通jar

1. 删除application.yml
2. 删除Application类(包含main方法的类)
3. 删除springboot maven plugin

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <includeSystemScope>true</includeSystemScope>
    </configuration>
    <!--指定打包时候将依赖一起打包-->
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

4. 添加普通打包插件

```markup
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>11</source>
        <target>11</target>
        <skip>true</skip>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <configuration>
        <encoding>UTF-8</encoding>
    </configuration>
</plugin>
```

5. 修改nacos配置，去掉数据库等和serviceA重复的内容

### serviceA引入serviceB

1. 添加serviceB jar依赖
2. componentScan添加serviceB 根路径 (@ComponentScan)
3. mapper添加serviceB mapper根路径 (@MapperScan)
4. application.yml 引入 serviceB的nacos配置

```
spring:
  # ...
  config:
    import:
      - optional:nacos:serviceB.yml?group=DEFAULT_GROUP&refreshEnabled=true
```

### serviceB路径转发

首先在serviceB中添加拦截器，将/serviceA/serviceB/xxx 转发到/serviceB/xxx

```java
@Slf4j
public class ForwardInterceptor implements HandlerInterceptor {

    private String prefix;

    public ForwardInterceptor(String prefix) {
        this.prefix = prefix;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 在请求之前添加统一的前缀
        String requestURI = request.getRequestURI();
        String newURI = requestURI.replaceFirst(prefix + "/serviceB", ""); 
        log.debug("ForwardInterceptor.preHandle, prefix={}, requestURI: {}, newURI={}", prefix, request.getRequestURI(), newURI);
        request.getRequestDispatcher(newURI).forward(request, response);
        return false;
    }
}

@Slf4j
@Configuration
public class InfInterceptorConfig implements WebMvcConfigurer {

    @Value("${spring.application.name}")
    private String application;

    @Value("${server.servlet.context-path}")
    private String contextPath;

    public static final String STATIC_CONTENT = "/static/file/**";
    private static final String ORIGIN_APPLICATION = "serviceB";

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 这里做兼容，如果serviceB回退为spring boot fat jar时，不做任何处理
        if (ORIGIN_APPLICATION.equalsIgnoreCase(application)) {
            return;
        }

        log.info("forward interceptor add, application={}, context-path={}", application, contextPath);
        // 配置需要的转发路径
        registry.addInterceptor(new ForwardInterceptor(contextPath))
                .addPathPatterns("/serviceB/**"); 
    }
}

```

### 网关转发

```yaml
spring:
    gateway:
      routes:
        - id: serviceA
          uri: lb://serviceA # serviceA
          predicates:
            - Path=/api/serviceA/**
          filters:
            - StripPrefix=1
        - id: serviceB
          uri: lb://serviceA # 注意这里需要改成serviceB
          predicates:
            - Path=/api/serviceB/**
          filters:
            # 这里需要做一个url rewrite
            - RewritePath=/api/serviceB/?(?<segment>.*), /api/ticket/serviceB/$\{segment}
            - StripPrefix=1 
```

### FeignClient改造

```java
@FeignClient(name = "serviceA", contextId = "serviceB", path = "/serviceA", fallbackFactory = InfrastructureFeignClientFallbackFactory.class)
public interface ServiceBFeignClient {
    // ...
}
```

* name：服务提供方id
* contextId：beanId
* path：因为url都变成为/serviceA/serviceB/\*\*了，所以这里需要加上/serviceA统一前缀

## 部署升级

1、升级serviceA

2、serviceB调用方一次升级

3、gateway转发规则更新，重启

4、下掉serviceB
