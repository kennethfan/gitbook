# spring cloud手动创建feign client

## 背景

spring cloud 默认或者主流的服务间调用方式是通过feign走的http调用；在某些场景下，服务提供方发布的feign client可能并不适用，需要稍微调整下，因此需要有时候需要调用方手动构造一个feign client

本文笔记记录一下

```java
 private static final String SERVICE_NAME = "service-xxx";
 public static final String API_NAME = "XxxApi";
  
 private String xxxServiceUrl;

 @Autowired
 private ApplicationContext applicationContext;

 private FeignClientBuilder feignClientBuilder;

 @Override
 public void afterPropertiesSet() throws Exception {
   feignClientBuilder = new FeignClientBuilder(applicationContext);
 }

 @Bean(API_NAME)
 public AgencyApi orgAgencyApi() {
   return getOrgFeignClient(AgencyApi.class, API_NAME, AgencyApi.PATH);
 }

 private <T> T getOrgFeignClient(Class<T> type, String name, String path) {
   return this.feignClientBuilder.forType(type, name)
       .path(path)
       .url(xxxServiceUrl)
       .contextId(SERVICE_NAME)
       .build();
 }

```
