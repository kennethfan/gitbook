# redis动态添加从库

启动从库，并连接

```bash
redis-cli -h 127.0.0.1 -a '密码'
```

连接从库

```bash
slaveof <host> <port>
```

确认同步信息

```bash
info replication
```

结果如下

![](<../.gitbook/assets/image (1) (1) (1).png>)

此时发现，数据并没有同步到从库；master\_link\_status:down

此时需要查看redis日志排查

有一种情况是主库设置了密码，从库在同步时需要设置密码

```bash
config set masterauth <密码>
```

操作完再确认状态，此时已经ok

将配置同步到配置文件

```bash
config rewrite
```
