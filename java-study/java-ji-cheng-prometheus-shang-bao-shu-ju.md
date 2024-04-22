---
description: java集成prometheus上报数据
---

# java集成prometheus上报数据

## 背景

公司业务发展越来越快，之前缺少的监控，这一块一直没有跟上

经过一番调研，决定使用prometheus+grafana实现监控报警

对于服务器资源、mysql、redis，prometheus官方都有推荐的exporter，因为对于系统资源等常规监控基本都可以实现

但是还有一类和业务相关的指标，常规的exporter没法直接处理，因此需要在应用程序中自行埋点，因此需要prometheus能够从应用程序采集数据

## 目标

解决prometheus从应用程序采集数据问题

## 现状

应用程序目前使用springboot框架，且已经集成了spring-boot-starter-actuator；actuator本身支持metrics，因此直接集成相关的jar即可

```markup
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>prometheus-metrics-core</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>prometheus-metrics-instrumentation-jvm</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.0.0</version>
</dependency>
```

&#x20;对应配置修改

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
      business: {具体业务} 
```
