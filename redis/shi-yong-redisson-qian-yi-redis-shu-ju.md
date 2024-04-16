---
description: 使用redisson迁移redis数据
---

# 使用redisson迁移redis数据

java代码

```java
import org.redisson.Redisson;
import org.redisson.api.RType;
import org.redisson.api.RedissonClient;
import org.redisson.client.codec.StringCodec;
import org.redisson.config.Config;

import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class RedisMigrate {

    public static void main(String[] args) {
    

        RedissonClient sourceRedis = createRedisClient(
                getProperty("redis.source.host"),
                getPropertyInteger("redis.source.port"),
                getProperty("redis.source.password"),
                getPropertyInteger("redis.source.database")
        );

        RedissonClient destRedis = createRedisClient(
                getProperty("redis.dest.host"),
                getPropertyInteger("redis.dest.port"),
                getProperty("redis.dest.password"),
                getPropertyInteger("redis.dest.database")
        );

        doMigrate(sourceRedis, destRedis);
    }

    private static String getProperty(String key) {
        String property = System.getProperty(key);

        if (property == null) {
            throw new RuntimeException("property " + key + " is null");
        }

        return property;
    }

    private static int getPropertyInteger(String key) {
        return Integer.parseInt(getProperty(key));
    }

    private static RedissonClient createRedisClient(String host, int port, String password, int database) {
        Config config = new Config();
        config.useSingleServer()
                .setAddress("redis://" + host + ":" + port)
                .setPassword(password.length() == 0 ? null : password)
                .setDatabase(database);

        return Redisson.create(config);
    }

    private static void doMigrate(RedissonClient source, RedissonClient dest) {
        Set<String> types = new HashSet<>();
        for (String key : source.getKeys().getKeys()) {
            RType type = source.getKeys().getType(key);
            types.add(type.name());
            dispatch(source, dest, key, type);
        }

        System.out.println("all types is " + types);
    }

    private static void dispatch(RedissonClient source, RedissonClient dest, String key, RType type) {
        System.out.println("dispatch key is " + key + ", type is " + type.name());
        switch (type) {
            case OBJECT:
                doMigrateObject(source, dest, key);
                break;
            case LIST:
                doMigrateList(source, dest, key);
                break;
            case MAP:
                doMigrateMap(source, dest, key);
                break;
            case SET:
                doMigrateSet(source, dest, key);
                break;
            case ZSET:
                doMigrateZSet(source, dest, key);
                break;
            default:
                System.out.println("dispatch key " + key + " type not support");
        }
    }

    private static void doMigrateObject(RedissonClient source, RedissonClient dest, String key) {
        dest.getBucket(key, StringCodec.INSTANCE).set(source.getBucket(key, StringCodec.INSTANCE).get());
        long expire = source.getKeys().remainTimeToLive(key);
        if (expire > 0L) {
            dest.getKeys().expire(key, expire, TimeUnit.MILLISECONDS);
        }
    }

    private static void doMigrateList(RedissonClient source, RedissonClient dest, String key) {
        dest.getList(key, StringCodec.INSTANCE).addAll(source.getList(key, StringCodec.INSTANCE));
        long expire = source.getKeys().remainTimeToLive(key);
        if (expire > 0L) {
            dest.getKeys().expire(key, expire, TimeUnit.MILLISECONDS);
        }
    }

    private static void doMigrateMap(RedissonClient source, RedissonClient dest, String key) {
        dest.getMap(key, StringCodec.INSTANCE).putAll(source.getMap(key, StringCodec.INSTANCE));
        long expire = source.getKeys().remainTimeToLive(key);
        if (expire > 0L) {
            dest.getKeys().expire(key, expire, TimeUnit.MILLISECONDS);
        }
    }

    private static void doMigrateSet(RedissonClient source, RedissonClient dest, String key) {
        dest.getSet(key, StringCodec.INSTANCE).addAll(source.getSet(key, StringCodec.INSTANCE));
        long expire = source.getKeys().remainTimeToLive(key);
        if (expire > 0L) {
            dest.getKeys().expire(key, expire, TimeUnit.MILLISECONDS);
        }
    }

    private static void doMigrateZSet(RedissonClient source, RedissonClient dest, String key) {
        source.getScoredSortedSet(key, StringCodec.INSTANCE).entryRange(0, -1)
                .forEach(entry -> dest.getScoredSortedSet(key, StringCodec.INSTANCE).add(entry.getScore(), entry.getValue()));
        long expire = source.getKeys().remainTimeToLive(key);
        if (expire > 0L) {
            dest.getKeys().expire(key, expire, TimeUnit.MILLISECONDS);
        }
    }
}

```
