# mysql创建用户并授权

```sql
CREATE USER 'app_user_dev'@'192.168.%' IDENTIFIED WITH  mysql_native_password BY '<密码>';  -- 创建用户

grant all privileges on <database>.* to  'app_user_dev'@'192.168.%'; -- 授权
```

