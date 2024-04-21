# mysql主从同步配置

## 主库

```sql
CREATE USER 'slave' @'%' IDENTIFIED BY '<密码>';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave' @'%';

FLUSH PRIVILEGES;

SHOW MASTER STATUS;
```

## 从库

```sql
CHANGE MASTER TO
MASTER_HOST='<主库host>',
MASTER_PORT=<主库端口>,
MASTER_USER='slave',
MASTER_PASSWORD='<密码>',
MASTER_LOG_FILE='mysql-bin.000002', # 主库show master status 会显示
MASTER_LOG_POS=154; # 主库show master status 会显示

SHOW SLAVE STATUS;
```
