---
description: dynamic-datasource实现数据读写分离
---

# dynamic-datasource实现数据读写分离

## 背景

随着业务流量越来越大，所有数据库请求都访问mysql主库，给主库造成了较大的压力

## 目标

减轻mysql主库压力

## 现状

当前数据库部署架构为一主一从，从库只是单纯作为备份，没有承接线上流量，资源有闲置

## 方案

### 方案对比

通常会有其中思路

#### 分库

分库的思路是讲数据库中表根据业务紧密程度拆分成几个不同的库，不同的表访问不同的库，不同的库分布在不同的节点，从而可以减轻单个库的压力，是一种横向扩展的思路，类似的还有分表，这里不做介绍

分库通常会涉及到系统的改造，比如原先在一个库的表，一个事务就可以保证，拆分成多个库之后，原先一个事务可以完成的，现在不能保证正确，通常涉及到分布式事务或者BASE等，改造成本比较高

#### 读写分离

读写分离也是业界常用的方案之一，思路是在数据库主从同步延迟可以接受的范围内，讲一部分查询请求分流到数据库从库，从来降低主库压力

考虑到目前的数据容量以及改造成本，现阶段先采用读写分离的方式

### 实施

因为系统已经集成了dynamic-datasource介入了多个不同的数据源，因此方案基于此作

先了解下dynamic-datasource核心类

首先是com.baomidou.dynamic.datasource.spring.boot.autoconfigure.DynamicDataSourceAutoConfiguration

```java
@Slf4j
@Configuration
@EnableConfigurationProperties(DynamicDataSourceProperties.class)
@AutoConfigureBefore(value = DataSourceAutoConfiguration.class, name = "com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure")
@Import(value = {DruidDynamicDataSourceConfiguration.class, DynamicDataSourceCreatorAutoConfiguration.class})
@ConditionalOnProperty(prefix = DynamicDataSourceProperties.PREFIX, name = "enabled", havingValue = "true", matchIfMissing = true)
public class DynamicDataSourceAutoConfiguration implements InitializingBean {

    private final DynamicDataSourceProperties properties;

    private final List<DynamicDataSourcePropertiesCustomizer> dataSourcePropertiesCustomizers;

    public DynamicDataSourceAutoConfiguration(
            DynamicDataSourceProperties properties,
            ObjectProvider<List<DynamicDataSourcePropertiesCustomizer>> dataSourcePropertiesCustomizers) {
        this.properties = properties;
        this.dataSourcePropertiesCustomizers = dataSourcePropertiesCustomizers.getIfAvailable();
    }

    @Bean
    public DynamicDataSourceProvider ymlDynamicDataSourceProvider() {
        return new YmlDynamicDataSourceProvider(properties.getDatasource());
    }

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        DynamicRoutingDataSource dataSource = new DynamicRoutingDataSource();
        dataSource.setPrimary(properties.getPrimary());
        dataSource.setStrict(properties.getStrict());
        dataSource.setStrategy(properties.getStrategy());
        dataSource.setP6spy(properties.getP6spy());
        dataSource.setSeata(properties.getSeata());
        return dataSource;
    }

    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    @Bean
    @ConditionalOnProperty(prefix = DynamicDataSourceProperties.PREFIX + ".aop", name = "enabled", havingValue = "true", matchIfMissing = true)
    public Advisor dynamicDatasourceAnnotationAdvisor(DsProcessor dsProcessor) {
        DynamicDatasourceAopProperties aopProperties = properties.getAop();
        DynamicDataSourceAnnotationInterceptor interceptor = new DynamicDataSourceAnnotationInterceptor(aopProperties.getAllowedPublicOnly(), dsProcessor);
        DynamicDataSourceAnnotationAdvisor advisor = new DynamicDataSourceAnnotationAdvisor(interceptor, DS.class);
        advisor.setOrder(aopProperties.getOrder());
        return advisor;
    }

    // ...

    @Bean
    @ConditionalOnMissingBean
    public DsProcessor dsProcessor(BeanFactory beanFactory) {
        DsHeaderProcessor headerProcessor = new DsHeaderProcessor();
        DsSessionProcessor sessionProcessor = new DsSessionProcessor();
        DsSpelExpressionProcessor spelExpressionProcessor = new DsSpelExpressionProcessor();
        spelExpressionProcessor.setBeanResolver(new BeanFactoryResolver(beanFactory));
        headerProcessor.setNextProcessor(sessionProcessor);
        sessionProcessor.setNextProcessor(spelExpressionProcessor);
        return headerProcessor;
    }

    // ...
}

```

该Configuration定义了几个bean

* ymlDynamicDataSourceProvider，用于创建实际的数据库如druid/c3p等，代码比较简单这里忽略
* dataSource，实际类型是DynamicRoutingDataSource，是对众多dataSource的封装，接下来会细讲
* dynamicDatasourceAnnotationAdvisor，主要功能就是解析接口或者方法上的@DS注解并将值并放入线程上下文（DynamicDataSourceContextHolder）中

接下来看DynamicRoutingDataSource类

先看afterPropertiesSet

```java
@Override
public void afterPropertiesSet() throws Exception {
	// 检查开启了配置但没有相关依赖
	checkEnv();
	// 添加并分组数据源
	Map<String, DataSource> dataSources = new HashMap<>(16);
	for (DynamicDataSourceProvider provider : providers) {
		dataSources.putAll(provider.loadDataSources());
	}
	for (Map.Entry<String, DataSource> dsItem : dataSources.entrySet()) {
		addDataSource(dsItem.getKey(), dsItem.getValue());
	}
	// 检测默认数据源是否设置
	if (groupDataSources.containsKey(primary)) {
		log.info("dynamic-datasource initial loaded [{}] datasource,primary group datasource named [{}]", dataSources.size(), primary);
	} else if (dataSourceMap.containsKey(primary)) {
		log.info("dynamic-datasource initial loaded [{}] datasource,primary datasource named [{}]", dataSources.size(), primary);
	} else {
		log.warn("dynamic-datasource initial loaded [{}] datasource,Please add your primary datasource or check your configuration", dataSources.size());
	}
}
```

主要是讲所有的实际数据库连接放到两个map

* Map\<String, DataSource>，最简单的分组，key是配置的数据库name，value是具体的数据库
* Map\<String, GroupDataSource>，将配置的数据库name按照下划线分割之后，取第一个分组

代码如下

```java
/**
 * 添加数据源
 *
 * @param ds         数据源名称
 * @param dataSource 数据源
 */
public synchronized void addDataSource(String ds, DataSource dataSource) {
	DataSource oldDataSource = dataSourceMap.put(ds, dataSource);
	// 新数据源添加到分组
	this.addGroupDataSource(ds, dataSource);
	// 关闭老的数据源
	if (oldDataSource != null) {
		closeDataSource(ds, oldDataSource);
	}
	log.info("dynamic-datasource - add a datasource named [{}] success", ds);
}

/**
 * 新数据源添加到分组
 *
 * @param ds         新数据源的名字
 * @param dataSource 新数据源
 */
private void addGroupDataSource(String ds, DataSource dataSource) {
	if (ds.contains(UNDERLINE)) {
		String group = ds.split(UNDERLINE)[0];
		GroupDataSource groupDataSource = groupDataSources.get(group);
		if (groupDataSource == null) {
			try {
				groupDataSource = new GroupDataSource(group, strategy.getDeclaredConstructor().newInstance());
				groupDataSources.put(group, groupDataSource);
			} catch (Exception e) {
				throw new RuntimeException("dynamic-datasource - add the datasource named " + ds + " error", e);
			}
		}
		groupDataSource.addDatasource(ds, dataSource);
	}
}
```

接下来看下AbstractRoutingDataSource，是DynamicRoutingDataSource的基类

```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource {

    /**
     * 抽象获取连接池
     *
     * @return 连接池
     */
    protected abstract DataSource determineDataSource();

    @Override
    public Connection getConnection() throws SQLException {
        String xid = TransactionContext.getXID();
        if (StringUtils.isEmpty(xid)) {
            return determineDataSource().getConnection();
        } else {
            String ds = DynamicDataSourceContextHolder.peek();
            ds = StringUtils.isEmpty(ds) ? "default" : ds;
            ConnectionProxy connection = ConnectionFactory.getConnection(ds);
            return connection == null ? getConnectionProxy(ds, determineDataSource().getConnection()) : connection;
        }
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        String xid = TransactionContext.getXID();
        if (StringUtils.isEmpty(xid)) {
            return determineDataSource().getConnection(username, password);
        } else {
            String ds = DynamicDataSourceContextHolder.peek();
            ds = StringUtils.isEmpty(ds) ? "default" : ds;
            ConnectionProxy connection = ConnectionFactory.getConnection(ds);
            return connection == null ? getConnectionProxy(ds, determineDataSource().getConnection(username, password))
                    : connection;
        }
    }
    // ...
}
```

getConnection方法可以看到动态切换数据的逻辑，具体逻辑留给了子类实现即determineDataSource方法

接下来，我们DynamicDataSourceAutoConfiguration.determineDataSource方法

```java
@Override
public DataSource determineDataSource() {
    String dsKey = DynamicDataSourceContextHolder.peek();
    return getDataSource(dsKey);
}

/**
 * 获取数据源
 *
 * @param ds 数据源名称
 * @return 数据源
 */
public DataSource getDataSource(String ds) {
    if (StringUtils.isEmpty(ds)) {
        return determinePrimaryDataSource();
    } else if (!groupDataSources.isEmpty() && groupDataSources.containsKey(ds)) {
        log.debug("dynamic-datasource switch to the datasource named [{}]", ds);
        return groupDataSources.get(ds).determineDataSource();
    } else if (dataSourceMap.containsKey(ds)) {
        log.debug("dynamic-datasource switch to the datasource named [{}]", ds);
        return dataSourceMap.get(ds);
    }
    if (strict) {
        throw new CannotFindDataSourceException("dynamic-datasource could not find a datasource named" + ds);
    }
    return determinePrimaryDataSource();
}

private DataSource determinePrimaryDataSource() {
    log.debug("dynamic-datasource switch to the primary datasource");
    DataSource dataSource = dataSourceMap.get(primary);
    if (dataSource != null) {
        return dataSource;
    }
    GroupDataSource groupDataSource = groupDataSources.get(primary);
    if (groupDataSource != null) {
        return groupDataSource.determineDataSource();
    }
    throw new CannotFindDataSourceException("dynamic-datasource can not find primary datasource");
}
```

可以发现，如果从上下文可以获取到ds，将根据ds选择数据库，如果没有ds，将直接根据配置的primary选择数据库

还记得上文中提到的两个map吗，可以发现如果ds命中第一map时，返回的数据库是固定的，也就无法实现所谓的读写分离；除非每个方法都手动通过@DS注解指定，这种方式不友好；当第一个map不命中且第二个map命中时，实现轻松在多个数据库之间动态切换，且不需要在方法上指定@DS注解

因此第一个关键点就是要对 <mark style="color:red;">数据库进行分组，配置方式如下</mark>

```yaml
spring:
  datasource:
    dynamic:
      datasource:
        group1_master:
          # 具体数据库配置
        group1_slave1:
          # 具体数据库配置
        group1_slave2:
          # 具体数据库配置
        group2_master:
          # 具体数据库配置
        group2_slave1:
          # 具体数据库配置
        group2_slave2:
          # 具体数据库配置
```

如果走到分组，具体分组逻辑是GroupDataSource.determineDataSource方法

```java
public String determineDsKey() {
    return dynamicDataSourceStrategy.determineKey(new ArrayList<>(dataSourceMap.keySet()));
}

public DataSource determineDataSource() {
    return dataSourceMap.get(determineDsKey());
}
```

可以看到，具体选择分组中的那个数据库，是根据dynamicDataSourceStrategy来决定的，因此我们只需要实现一个dynamicDataSourceStrategy的实现能够根据读写请求选择不同的ds即可。

策略基本如下：

1、普通读请求，走从库或者主库都可以

2、普通写请求，走主库

3、强制指定了走主库的请求，走主库

4、事务执行，强制走主库

接下来难点在于

1、如何判断是读请求还是写请求

2、如何判断强制走主库

3、对于事物，比较特殊，在事务开启之前就需要切到主库然后获取数据库连接

一个一个解决

首先判断是否读请求还是写请求，这个需要使用mybatis拦截器，拦截Executor方法

```java
public interface Executor {
    ResultHandler NO_RESULT_HANDLER = null;

    int update(MappedStatement var1, Object var2) throws SQLException;

    <E> List<E> query(MappedStatement var1, Object var2, RowBounds var3, ResultHandler var4, CacheKey var5, BoundSql var6) throws SQLException;

    <E> List<E> query(MappedStatement var1, Object var2, RowBounds var3, ResultHandler var4) throws SQLException;
   // ...
}
```

通过第一个参数MappedStatement可以判断

```java
SqlCommandType.SELECT == MappedStatement.getSqlCommandType()
```

接下来就是如何判断强制走主库了，这个需要自定义实现，通常是使用注解+AOP方式

所以整体实现比较简单

首先定义上下文，保存是否走主库标志

```java
public class ForceMasterContext {

    private static final ThreadLocal<Boolean> FORCE_MASTER = new ThreadLocal<>();

    public static void setForceMaster(boolean forceMaster) {
        FORCE_MASTER.set(forceMaster);
    }

    public static boolean isForceMaster() {
        return FORCE_MASTER.get() != null && FORCE_MASTER.get();
    }

    public static void clearForceMaster() {
        FORCE_MASTER.remove();
    }
}

```

接下来实现策略，根据标志位选择主库或者从库

```java
@Slf4j
public class ForceMasterDynamicDataSourceStrategy extends LoadBalanceDynamicDataSourceStrategy {

    private static final String MASTER_SUFFIX = "_master";

    @Override
    public String determineKey(List<String> dsNames) {
        String ds = null;
        if (ForceMasterContext.isForceMaster()) {
            ds = Optional.ofNullable(dsNames)
                    .orElse(new ArrayList<>())
                    .stream()
                    .filter(StringUtils::isNotBlank)
                    .filter(o -> o.endsWith(MASTER_SUFFIX))
                    .findFirst()
                    .orElse(null);
        }

        if (ds == null) {
            ds = super.determineKey(dsNames);
        }

        return ds;
    }
}
```

需要修改配置，指定spring.datasource.dynamic.strategy=Xxx.class才会生效

定义mybatis拦截器，设置

```java
@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
), @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}
), @Signature(
        type = Executor.class,
        method = "update",
        args = {MappedStatement.class, Object.class}
)})
@Slf4j
@Component
public class ForceMasterPlugin implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 事务操作，正常不应该走到这里
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            return invocation.proceed();
        }
        
        // 已经强制走到主库了
        if (ForceMasterContext.isForceMaster()) {
            return invocation.proceed();
        }

        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        // 非事务中的查询操作直接返回
        if (SqlCommandType.SELECT == ms.getSqlCommandType()) {
            return invocation.proceed();
        }

        try {
            ForceMasterContext.setForceMaster(true);
            return invocation.proceed();
        } finally {
            ForceMasterContext.clearForceMaster();
        }
    }

    @Override
    public Object plugin(Object target) {
        return target instanceof Executor ? Plugin.wrap(target, this) : target;
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```

接下来自定义强制走主库注解

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ForceMaster {
}
```

定义Aop

```java
@Configuration
@Aspect
@Slf4j
public class ForceMasterAspectConfiguration {

    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    @Bean
    public Advisor forceMasterAdvisor() {
        MethodInterceptor interceptor = invocation -> {
            if (ForceMasterContext.isForceMaster()) {
                return invocation.proceed();
            }

            try {
                ForceMasterContext.setForceMaster(true);
                return invocation.proceed();
            } finally {
                ForceMasterContext.clearForceMaster();
            }
        };
        return new DynamicDataSourceAnnotationAdvisor(interceptor, ForceMaster.class);
    }
}
```

最后如何判断事务即将开启，spring-tx开启事务在AbstractPlatformTransactionManager.getTransaction方法，代码如下

```java
@Override
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // Use defaults if no transaction definition given.
    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

    Object transaction = doGetTransaction();
    boolean debugEnabled = logger.isDebugEnabled();

    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(def, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
        }
        try {
            return startTransaction(def, transaction, debugEnabled, suspendedResources);
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + def);
        }
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}

/**
 * Start a new transaction.
 */
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
        boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    DefaultTransactionStatus status = newTransactionStatus(
            definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
    doBegin(transaction, definition);
    prepareSynchronization(status, definition);
    return status;
}
```

最终会走到doBegin方法，具体实现类在DataSourceTransactionManager中

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionManager.DataSourceTransactionObject txObject = (DataSourceTransactionManager.DataSourceTransactionObject)transaction;
    Connection con = null;

    try {
        if (!txObject.hasConnectionHolder() || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = this.obtainDataSource().getConnection();
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }

            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }

            con.setAutoCommit(false);
        }

        this.prepareTransactionalConnection(con, definition);
        txObject.getConnectionHolder().setTransactionActive(true);
        int timeout = this.determineTimeout(definition);
        if (timeout != -1) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(this.obtainDataSource(), txObject.getConnectionHolder());
        }

    } catch (Throwable var7) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, this.obtainDataSource());
            txObject.setConnectionHolder((ConnectionHolder)null, false);
        }

        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", var7);
    }
}
```

可以看到doBegin方法中获取了数据连接(Connection对象），结合上面的两个方法，可以发现两点

1、在获取数据库连接之前没有任何的回调和埋点，也就是说无法通过添加钩子在做额外逻辑，也不能通过上下文去判断

2、方法要么是protected，要么是final修饰，也就是说不能通过AOP来扩展

因此只剩下一种方法，通过集成去扩展，且需要修改默认的PlatformTransactionManager实现类

代码如下

```java
@Component
public class ForceMasterDataSourceTransactionManager extends DataSourceTransactionManager {

    public ForceMasterDataSourceTransactionManager(DataSource dataSource, ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
        super(dataSource);

        TransactionManagerCustomizers customizers = transactionManagerCustomizers.getIfAvailable();
        if (customizers != null) {
            customizers.customize(this);
        }
    }

    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        if (!ForceMasterContext.isForceMaster()) {
            ForceMasterContext.setForceMaster(true);
        }

        super.doBegin(transaction, definition);
    }


    @Override
    protected void doCleanupAfterCompletion(Object transaction) {
        try {
            super.doCleanupAfterCompletion(transaction);
        } finally {
            ForceMasterContext.clearForceMaster();
        }
    }
}
```
