# spring cloud gateway + nacos + sentinel实现网关限流

## 背景

项目中有一些接口是非常消耗资源和耗时的，出于保护系统的目的，需要限流

## 目标

实现网关接口限流，保护系统

## 方案

项目采用了spring cloud gateway作为网关，nacos作为配置中心和注册中心

sentinel是开源的比较优秀的限流组件，此文采用sentinel作为限流实现

下面重点介绍sentinel集成

### maven坐标

```markup
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
 <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
```

### 限流组件相关配置

```yaml
spring:
  cloud:
    sentinel:
      eager: true
      scg:
        fallback:
          mode: response
          response-body: '{"code":429, "msg": "系统繁忙，请稍后重试"}'
          response-status: 200
      datasource:
        gw-flow: # 此处使用nacos作为数据源，参考com.alibaba.cloud.sentinel.datasource.config.DataSourcePropertiesConfiguration
          nacos: # 参考com.alibaba.cloud.sentinel.datasource.config.NacosDataSourceProperties
            server-addr: ${spring.cloud.nacos.discovery.server-addr}
            username: ${spring.cloud.nacos.discovery.username}
            password: ${spring.cloud.nacos.discovery.password}
            namespace: ${spring.cloud.nacos.discovery.namespace}
            data-id: sentinel-gw-flow.json
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: gw-flow
        api-group:
          nacos:
            server-addr: ${spring.cloud.nacos.discovery.server-addr}
            username: ${spring.cloud.nacos.discovery.username}
            password: ${spring.cloud.nacos.discovery.password}
            namespace: ${spring.cloud.nacos.discovery.namespace}
            data-id: sentinel-api-group.json
            group-id: DEFAULT_GROUP
            data-type: json
            rule-type: gw-api-group
```

### 限流配置

配置参考com.alibaba.cloud.sentinel.datasource.RuleType

sentinel-gw-flow.json，参考com.alibaba.csp.sentinel.adapter.gateway.common.rule.GatewayFlowRule

```json
[
    {
        "resource":"getTicket",
        "resourceMode":1,
        "degrade":0,
        "count":1,
        "controlBehavior":3
    }
]
```

sentinel-api-group.json，参考com.alibaba.csp.sentinel.adapter.gateway.common.api.ApiDefinition

```json
[
    {
        "apiName":"getTicket",
        "predicateItems":[
            {
                "pattern":"/api/ticket/adWelfare/givenTicket"
            }
        ]
    }
]
```
