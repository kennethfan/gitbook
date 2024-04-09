---
description: redis主从架构利用lettuce实现读写分离
---

# redis主从架构利用lettuce实现读写分离

## 背景

随着业务流量越来越大，原先所有流量都访问redis主库，给主库造成了很大的压力

## 目标

在不影响业务的前提下，减轻redis主库压力

## 现状

当前redis的部署架构是一主一从，从库只是承担了备份的角色，资源有很大的闲置

## 方案

如果从库也能承担一部分线上流量，那么主库的压力自然就会减轻；方案理论上可行

### 问题

项目使用的lettuce + spring-boot-starter-data-redis做redis访问

lettuce本身是支持主从模式的访问的，奈何spring-boot-starter-data-redis对于redis哨兵和redis集群模式都有很好的支持，对于主从没有支持

### 代码分析

#### 入口

org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    // ...
}
```

只看上面的import部分，默认会LettuceConnectionConfiguration会生效，接着看下这个文件

```java
@Bean
@ConditionalOnMissingBean(RedisConnectionFactory.class)
LettuceConnectionFactory redisConnectionFactory(
		ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
		ClientResources clientResources) {
	LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources,
			getProperties().getLettuce().getPool());
	return createLettuceConnectionFactory(clientConfig);
}

private LettuceConnectionFactory createLettuceConnectionFactory(LettuceClientConfiguration clientConfiguration) {
	if (getSentinelConfig() != null) {
		return new LettuceConnectionFactory(getSentinelConfig(), clientConfiguration);
	}
	if (getClusterConfiguration() != null) {
		return new LettuceConnectionFactory(getClusterConfiguration(), clientConfiguration);
	}
	return new LettuceConnectionFactory(getStandaloneConfig(), clientConfiguration);
}

private LettuceClientConfiguration getLettuceClientConfiguration(
		ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
		ClientResources clientResources, Pool pool) {
	LettuceClientConfigurationBuilder builder = createBuilder(pool);
	applyProperties(builder);
	if (StringUtils.hasText(getProperties().getUrl())) {
		customizeConfigurationFromUrl(builder);
	}
	builder.clientOptions(createClientOptions());
	builder.clientResources(clientResources);
	builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
	return builder.build();
}
```

redisConnectionFactory会创建最终的RedisConnectionFactory，再看createLettuceConnectionFactory方法，可以发现对哨兵(sentinel)和集群(cluster)是天然支持的，具体怎么配置这里先略过

我们再看LettuceConnectionFactory，这里重点关注afterPropertiesSet方法

```java
public void afterPropertiesSet() {
	this.client = this.createClient();
	this.connectionProvider = new LettuceConnectionFactory.ExceptionTranslatingConnectionProvider(this.createConnectionProvider(this.client, LettuceConnection.CODEC));
	this.reactiveConnectionProvider = new LettuceConnectionFactory.ExceptionTranslatingConnectionProvider(this.createConnectionProvider(this.client, LettuceReactiveRedisConnection.CODEC));
	if (this.isClusterAware()) {
		this.clusterCommandExecutor = new ClusterCommandExecutor(new LettuceClusterTopologyProvider((RedisClusterClient)this.client), new LettuceClusterNodeResourceProvider(this.connectionProvider), EXCEPTION_TRANSLATION);
	}

	this.initialized = true;
	if (this.getEagerInitialization() && this.getShareNativeConnection()) {
		this.initConnection();
	}
}
```

再看下createConnectionProvider方法

```java
private LettuceConnectionProvider createConnectionProvider(AbstractRedisClient client, RedisCodec<?, ?> codec) {
	if (this.pool != null) {
		return new LettucePoolConnectionProvider(this.pool);
	} else {
		LettuceConnectionProvider connectionProvider = this.doCreateConnectionProvider(client, codec);
		return (LettuceConnectionProvider)(this.clientConfiguration instanceof LettucePoolingClientConfiguration ? new LettucePoolingConnectionProvider(connectionProvider, (LettucePoolingClientConfiguration)this.clientConfiguration) : connectionProvider);
	}
}

protected LettuceConnectionProvider doCreateConnectionProvider(AbstractRedisClient client, RedisCodec<?, ?> codec) {
	ReadFrom readFrom = (ReadFrom)this.getClientConfiguration().getReadFrom().orElse((Object)null);
	if (this.isStaticMasterReplicaAware()) {
		List<RedisURI> nodes = (List)((RedisStaticMasterReplicaConfiguration)this.configuration).getNodes().stream().map((it) -> {
			return this.createRedisURIAndApplySettings(it.getHostName(), it.getPort());
		}).peek((it) -> {
			it.setDatabase(this.getDatabase());
		}).collect(Collectors.toList());
		return new StaticMasterReplicaConnectionProvider((RedisClient)client, codec, nodes, readFrom);
	} else {
		return (LettuceConnectionProvider)(this.isClusterAware() ? new ClusterConnectionProvider((RedisClusterClient)client, codec, readFrom) : new StandaloneConnectionProvider((RedisClient)client, codec, readFrom));
	}
}
```

最终会走到最后一行的最后一条语句new StandaloneConnectionProvider((RedisClient)client, codec, readFrom));

再看StandaloneConnectionProvider，重点看getConnection方法

```java
@Override
public <T extends StatefulConnection<?, ?>> T getConnection(Class<T> connectionType) {

	if (connectionType.equals(StatefulRedisSentinelConnection.class)) {
		return connectionType.cast(client.connectSentinel());
	}

	if (connectionType.equals(StatefulRedisPubSubConnection.class)) {
		return connectionType.cast(client.connectPubSub(codec));
	}

	if (StatefulConnection.class.isAssignableFrom(connectionType)) {

		return connectionType.cast(readFrom.map(it -> this.masterReplicaConnection(redisURISupplier.get(), it))
				.orElseGet(() -> client.connect(codec)));
	}

	throw new UnsupportedOperationException("Connection type " + connectionType + " not supported!");
}
```

当connectionType=StatefulConnection时，会走到masterReplicaConnection方法

```java
private StatefulRedisConnection masterReplicaConnection(RedisURI redisUri, ReadFrom readFrom) {

	StatefulRedisMasterReplicaConnection<?, ?> connection = MasterReplica.connect(client, codec, redisUri);
	connection.setReadFrom(readFrom);

	return connection;
}
```

继续往下追，会走到MasterReplica.connectAsyncSentinelOrAutodiscovery方法

```java
private static <K, V> CompletableFuture<StatefulRedisMasterReplicaConnection<K, V>> connectAsyncSentinelOrAutodiscovery(
		RedisClient redisClient, RedisCodec<K, V> codec, RedisURI redisURI) {

	LettuceAssert.notNull(redisClient, "RedisClient must not be null");
	LettuceAssert.notNull(codec, "RedisCodec must not be null");
	LettuceAssert.notNull(redisURI, "RedisURI must not be null");

	if (isSentinel(redisURI)) {
		return new SentinelConnector<>(redisClient, codec, redisURI).connectAsync();
	}

	return new AutodiscoveryConnector<>(redisClient, codec, redisURI).connectAsync();
}
```

最终走到AutodiscoveryConnector，重点关注initializeConnection方法

```java
private Mono<StatefulRedisMasterReplicaConnection<K, V>> initializeConnection(RedisCodec<K, V> codec,
		Tuple2<RedisURI, StatefulRedisConnection<K, V>> connectionAndUri) {

	ReplicaTopologyProvider topologyProvider = new ReplicaTopologyProvider(connectionAndUri.getT2(),
			connectionAndUri.getT1());

	MasterReplicaTopologyRefresh refresh = new MasterReplicaTopologyRefresh(redisClient, topologyProvider);
	MasterReplicaConnectionProvider<K, V> connectionProvider = new MasterReplicaConnectionProvider<>(redisClient, codec,
			redisURI, (Map) initialConnections);

	Mono<List<RedisNodeDescription>> refreshFuture = refresh.getNodes(redisURI);

	return refreshFuture.map(nodes -> {

		EventRecorder.getInstance().record(new MasterReplicaTopologyChangedEvent(redisURI, nodes));

		connectionProvider.setKnownNodes(nodes);

		MasterReplicaChannelWriter channelWriter = new MasterReplicaChannelWriter(connectionProvider,
				redisClient.getResources());

		StatefulRedisMasterReplicaConnectionImpl<K, V> connection = new StatefulRedisMasterReplicaConnectionImpl<>(
				channelWriter, codec, redisURI.getTimeout());

		connection.setOptions(redisClient.getOptions());

		return connection;
	});
}
```

可以发现返回的StatefulRedisMasterReplicaConnection对象中，channerWriter是MasterReplicaChannelWriter；重点看write方法，大部分redis操作都走到这个方法

```java
@Override
@SuppressWarnings("unchecked")
public <K, V, T> RedisCommand<K, V, T> write(RedisCommand<K, V, T> command) {

	LettuceAssert.notNull(command, "Command must not be null");

	if (closed) {
		throw new RedisException("Connection is closed");
	}

	if (isStartTransaction(command.getType())) {
		inTransaction = true;
	}

	Intent intent = inTransaction ? Intent.WRITE : getIntent(command.getType());
	CompletableFuture<StatefulRedisConnection<K, V>> future = (CompletableFuture) masterReplicaConnectionProvider
			.getConnectionAsync(intent);

	if (isEndTransaction(command.getType())) {
		inTransaction = false;
	}

	if (isSuccessfullyCompleted(future)) {
		writeCommand(command, future.join(), null);
	} else {
		future.whenComplete((c, t) -> writeCommand(command, c, t));
	}

	return command;
}
```

重点在CompletableFuture\<StatefulRedisConnection\<K, V>> future = (CompletableFuture) masterReplicaConnectionProvider .getConnectionAsync(intent);

```java
public CompletableFuture<StatefulRedisConnection<K, V>> getConnectionAsync(Intent intent) {

	if (debugEnabled) {
		logger.debug("getConnectionAsync(" + intent + ")");
	}

	if (readFrom != null && intent == Intent.READ) {
		List<RedisNodeDescription> selection = readFrom.select(new ReadFrom.Nodes() {

			@Override
			public List<RedisNodeDescription> getNodes() {
				return knownNodes;
			}

			@Override
			public Iterator<RedisNodeDescription> iterator() {
				return knownNodes.iterator();
			}

		});

		if (selection.isEmpty()) {
			throw new RedisException(String.format("Cannot determine a node to read (Known nodes: %s) with setting %s",
					knownNodes, readFrom));
		}

		try {

			Flux<StatefulRedisConnection<K, V>> connections = Flux.empty();

			for (RedisNodeDescription node : selection) {
				connections = connections.concatWith(Mono.fromFuture(getConnection(node)));
			}

			if (OrderingReadFromAccessor.isOrderSensitive(readFrom) || selection.size() == 1) {
				return connections.filter(StatefulConnection::isOpen).next().switchIfEmpty(connections.next()).toFuture();
			}

			return connections.filter(StatefulConnection::isOpen).collectList().map(it -> {
				int index = ThreadLocalRandom.current().nextInt(it.size());
				return it.get(index);
			}).switchIfEmpty(connections.next()).toFuture();
		} catch (RuntimeException e) {
			throw Exceptions.bubble(e);
		}
	}

	return getConnection(getMaster());
}
```

其中有一行需要注意；if (readFrom != null && intent == Intent.READ) 块中的内容

可以发现，当遇到读请求且readFrom不为空时，会有选择节点的策略

所以要实现读写分离，我们只需要保证走到这里时，readFrom是从从库选择节点的策略就行

其中内置的ReadFrom.REPLICA\_PREFERRED即可满足要求

接下来需要解决的是如何保证走到这里时readFrom不为空，实际上这里无法直接通过配置完成，具体可以看RedisProperties，里面没有定义这个字段，接下来，我们向上回溯可以发现

LettuceConnectionConfiguration.getLettuceClientConfiguration此时可以通过新增LettuceClientConfigurationBuilderCustomizer去设置readFrom，整个推理过程不在描述

因此，我们只需要增加一个LettuceClientConfigurationBuilderCustomizer实现即可

```java
@ConditionalOnProperty(value = "spring.redis.replica-preferred", havingValue = "true")
@Component
@Slf4j
public class RedisPreferredLettuceClientConfigurationBuilderCustomizer implements LettuceClientConfigurationBuilderCustomizer, InitializingBean {

    @Override
    public void customize(LettuceClientConfiguration.LettuceClientConfigurationBuilder clientConfigurationBuilder) {
        log.info("RedisPreferredLettuceClientConfigurationBuilderCustomizer.customize");
        clientConfigurationBuilder.readFrom(ReadFrom.REPLICA_PREFERRED);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        log.info("RedisPreferredLettuceClientConfigurationBuilderCustomizer inited");
    }
}
```

