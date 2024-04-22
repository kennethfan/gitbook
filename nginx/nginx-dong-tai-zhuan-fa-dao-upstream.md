---
description: nginx动态转发到upstream
---

# nginx动态转发到upstream

## 背景

最近在使用prometheus做监控，应用程序也集成了prometheus，因此有很多的数据采集是跟随应用程序走的，导致采集的地址有很多，且都是非公开的访问端口，因此prometheus没法直接从应用程序采集数据

## 目标

解决prometheus从应用程序采集指标问题

## 方案

内网端口不直接对外暴露，这个是不可以改的，因此只能通过可以暴露的一些端口来做转发

在nginx上这种方式是可以实现的，我们定义了一个转发的路径/dispatch，需要动态转发的都走到这里

然后定义了两个参数up和path：

up表示具体的服务提供方ip+端口

path表示具体的请求path

```
location /dispatch {
        allow 192.168.0.0/16; # 只允许内网访问
        deny all;
        
        set $metrics_up $arg_up;
        rewrite (.*)$ $arg_path break;

        proxy_set_header path $arg_up;
        proxy_set_header upstream $arg_up;
        proxy_pass http://$metrics_up;
}
```

因此如果访问http://mydomain.com/dispatch?path=/test/hello\&up=127.0.0.1:8080

实际会转发到http://127.0.0.1:8080/test/hello
