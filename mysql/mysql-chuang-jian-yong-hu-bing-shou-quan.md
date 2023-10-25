# mysql创建用户并授权

```
CREATE USER 'app_user_dev'@'192.168.%' IDENTIFIED WITH  mysql_native_password BY 'tUJgpb68tRoWZQ==';  -- 创建用户

grant all privileges on plant_client_dev.* to  'app_user_dev'@'192.168.%'; -- 授权
```

