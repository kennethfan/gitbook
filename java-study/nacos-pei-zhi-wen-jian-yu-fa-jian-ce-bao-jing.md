---
description: nacos配置文件语法检测&报警
---

# nacos配置文件语法检测&报警

## 背景 <a href="#nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-bei-jing" id="nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-bei-jing"></a>

项目使用nacos作为配置中心，nacos控制台对于语法检测功能比较弱，因此有时会发生配置文件格式不对导致项目重启或者发布时服务起不来的情况，排查起来也比较麻烦

## 目标 <a href="#nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-mu-biao" id="nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-mu-biao"></a>

实时检测出nacos配置文件的语法错误，并做出报警提示

## 方案 <a href="#nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-fang-an" id="nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-fang-an"></a>

| 方案               | 有点                    | 缺点                                               |
| ---------------- | --------------------- | ------------------------------------------------ |
| 扩展nacos控制台，控制台监测 | 1、一劳永逸，客户端无需改造        | <p>1、改造成本比较高</p><p>2、为未来升级nacos带来了潜在的风险和不兼容性</p> |
| 客户端实时监测          | 1、改动相对简单，插件化，可插拔，风险可控 | 1、如果有多语言客户端，改造的成本会比较高                            |

考虑到公司主要以java作为开发语言，因此采用第二种方案

### 详细设计 <a href="#nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-xiang-xi-she-ji" id="nacos-pei-zhi-wen-jian-yu-fa-jian-ce-bao-jing-xiang-xi-she-ji"></a>

以下方案基于spring-cloud-starter-alibaba-nacos-config

先介绍几个重要的class

#### com.alibaba.cloud.nacos.NacosConfigManager

```java

// 获取ConfigService
public ConfigService getConfigService() {
   if (Objects.isNull(service)) {
      createConfigService(this.nacosConfigProperties);
   }
   return service;
}
// 获取nacos全部配置，比如server-addr、namespace等
public NacosConfigProperties getNacosConfigProperties() {
   return nacosConfigProperties;
}
```

#### com.alibaba.nacos.api.config.ConfigService

<pre class="language-java"><code class="lang-java"><strong>// 这对配置单个配置文件添加监听器，可以监听到配置变更
</strong>void addListener(String dataId, String group, Listener listener) throws NacosException;
</code></pre>

com.alibaba.cloud.nacos.NacosConfigProperties

```java
public String getServerAddr() {
   return serverAddr;
}
 
public String getNamespace() {
   return namespace;
}
```

#### com.alibaba.cloud.nacos.NacosPropertySourceRepository

```java
// 获取所有的nacos配置文件
public static List<NacosPropertySource> getAll() {
   return new ArrayList<>(NACOS_PROPERTY_SOURCE_REPOSITORY.values());
}
```

#### com.alibaba.cloud.nacos.client.NacosPropertySource

```java
public String getGroup() {
   return this.group;
}
 
public String getDataId() {
   return dataId;
}
```

#### com.alibaba.nacos.api.config.listener.Listener&#x20;

```java
// 配置文件变更时触发
void receiveConfigInfo(final String configInfo);
```

所以方案也比较简单：在服务启动之后，拿到所有的配置文件，然后针对每个配置文件，分别添加listener，实现receiveConfigInfo方法，监听到变更时，针对configInfo做语法检查，如果检查失败打印日志并报警

下面上代码

```java
public class NacosSyntaxCheckListener implements ApplicationListener<ApplicationStartedEvent>, InitializingBean {
     
    private NacosConfigManager nacosConfigManager;
 
    private List<ConfigSyntaxErrorHandler> errorHandlerList;
 
    private Map<String, ConfigSyntaxChecker> checkerMap;
 
    private ExecutorService executor;
 
    // 构造器注入依赖
    public NacosSyntaxCheckListener(NacosConfigManager nacosConfigManager, List<ConfigSyntaxChecker> configSyntaxCheckerList, List<ConfigSyntaxErrorHandler> configSyntaxErrorHandlerList) {
        this.nacosConfigManager = nacosConfigManager;
 
        // 不同文件类型有不同的语法规则，这里根据文件扩展名来区分，通过List结构实现自动扩展
        this.checkerMap = new HashMap<>();
        Optional.ofNullable(configSyntaxCheckerList)
                .orElse(new ArrayList<>())
                .forEach(this::registerSyntaxChecker);
        log.info("NacosSyntaxCheckListener constructor, checkerMap.size={}", this.checkerMap.size());
 
        // 语法检查失败处理，通过List结构实现自动扩展
        this.errorHandlerList = Optional.ofNullable(configSyntaxErrorHandlerList).orElse(new ArrayList<>());
        log.info("NacosSyntaxCheckListener constructor, errorHandlerList.size={}", this.errorHandlerList.size());
 
        this.executor = Executors.newSingleThreadExecutor();
    }
 
    // 同一种类型多种扩展名(比如.yml和.yaml是一种语法)支持
    private void registerSyntaxChecker(ConfigSyntaxChecker configSyntaxChecker) {
        Optional.ofNullable(configSyntaxChecker.supportExtension())
                .orElse(new ArrayList<>())
                .forEach(extension -> this.checkerMap.put(extension, configSyntaxChecker));
    }
 
    @Override
    public void afterPropertiesSet() throws Exception {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> executor.shutdown()));
    }
 
    // 服务启动之后，获取所有配置，并添加监听器
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        com.alibaba.cloud.nacos.NacosPropertySourceRepository.getAll().forEach(this::addListener);
    }
 
    private void addListener(NacosPropertySource nacosPropertySource) {
        log.info("NacosSyntaxCheckListener.addListener begin, dataId={}, group={}",
                nacosPropertySource.getDataId(), nacosPropertySource.getGroup());
 
        String extension = FilenameUtils.getExtension(nacosPropertySource.getDataId());
        final ConfigSyntaxChecker checker = checkerMap.get(extension);
        if (extension == null) {
            log.info("NacosSyntaxCheckListener.addListener no checker support, dataId={}, group={}",
                    nacosPropertySource.getDataId(), nacosPropertySource.getGroup());
            return;
        }
 
        try {
            nacosConfigManager.getConfigService().addListener(nacosPropertySource.getDataId(), nacosPropertySource.getGroup(), new Listener() {
                @Override
                public Executor getExecutor() {
                    return executor;
                }
 
                @Override
                public void receiveConfigInfo(String configInfo) {
                    try {
                        log.info("NacosSyntaxCheckListener.receiveConfigInfo, dataId={}, group={}, config={}",
                                nacosPropertySource.getDataId(), nacosPropertySource.getGroup(), configInfo);
                        // 语法检查
                        checker.check(configInfo);
                        log.info("NacosSyntaxCheckListener.receiveConfigInfo, check success, dataId={}, group={}",
                                nacosPropertySource.getDataId(), nacosPropertySource.getGroup());
                    } catch (Exception e) {
                        NacosConfigProperties nacosConfigProperties = nacosConfigManager.getNacosConfigProperties();
                        errorHandlerList.forEach(handler -> {
                            // 语法检查失败之后做失败处理，比如打日志报警，或者发送钉钉/企微等方式
                            handler.onError(nacosConfigProperties, nacosPropertySource, e.getMessage());
                        });
                    }
                }
            });
        } catch (Exception e) {
            log.error("NacosSyntaxCheckListener.addListener error, dataId={}, group={}",
                    nacosPropertySource.getDataId(), nacosPropertySource.getGroup(), e);
        }
    }
}
 
 
public interface ConfigSyntaxChecker {
    // 支持的文件扩展名
    Collection<String> supportExtension();
    // 语法检测
    void check(String content);
}
 
// 内置的yaml格式语法检查，其他格式语法检查留作扩展，这里并未实现
public class YamlConfigSyntaxChecker implements ConfigSyntaxChecker {
 
    @Override
    public Collection<String> supportExtension() {
        return Sets.newHashSet("yml", "yaml");
    }
 
    @Override
    public void check(String content) {
        Yaml yaml = new Yaml();
        yaml.load(content);
    }
}
 
public interface ConfigSyntaxErrorHandler {
    // 语法检测失败之后的错误处理
    void onError(NacosConfigProperties nacosConfigProperties, NacosPropertySource nacosPropertySource, String errorMsg);
}
 
// 内置的错误处理，这里只打日志，其他诉求可以自定义接口实现扩展
public class LogConfigSyntaxErrorHandler implements ConfigSyntaxErrorHandler {
 
    @Override
    public void onError(NacosConfigProperties nacosConfigProperties, NacosPropertySource nacosPropertySource, String errorMsg) {
        log.error("nacos config content check error, server={}, namespace={}, dataId={}, group={}, msg={}",
                nacosConfigProperties.getServerAddr(), nacosConfigProperties.getNamespace(),
                nacosPropertySource.getDataId(), nacosPropertySource.getGroup(),
                errorMsg);
    }
}
```
