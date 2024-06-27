---
description: nginx正向代理https
---

# nginx正向代理https

## 背景

公司有项目因为是和外部合作，考虑到公司网络的不稳定，决定将服务上云，上云之后暴露出一些新的问题：原有的一部分后台服务（比如nacos后台，文件服务后台）不得不暴露在外网，这是非常危险的，因此需要对这部分服务做一些ip限制，因为办公网络并非走的专线，因此对ip限制带来了一些复杂度

## 目标

稳定访问ip，降低ip限制复杂度

## 方案

公司的生产服务器走的专线，出口ip比较固定，因此，如果能够通过生产服务器做一个代理访问到后台服务，此时访问后台服务的ip都是固定的出口ip，此时做ip限制就非常好做了。

经过调研，通过nginx做正向代理是非常容易的

现在https已经是基础要求了，所以nginx需要能够正向代理https；实际上nginx代理https也是可以得，不过需要安装特定模块，此处记录下步骤，基于centos 7.x

### 基础软件准备

```bash
yum -y install git patch gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

### nginx扩展模块安装

#### 代码clone

```bash
git clone https://github.com/chobits/ngx_http_proxy_connect_module.git
```

#### nginx(重)编译安装

```bash
cd /path/to/nginx-source # nginx源码目录

patch -p1 < /path/to/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_102101.patch # 打补丁，具体的使用哪个patch需要根据nginx版本确定

./configure ... --add-module=/path/to/ngx_http_proxy_connect_module # 如果是已经编译过 /path/to/nginx -V 可以查看原有编译参数

make && make install # 编译安装
```

#### 正向代理配置

```bash
server {
     listen                         8765;                #对外服务端口
     resolver                       223.6.6.6 ipv6=off;     #域名解析服务器
     proxy_connect;
     proxy_connect_allow            all;
     proxy_connect_connect_timeout  10s;
     proxy_connect_read_timeout     10s;
     proxy_connect_send_timeout     10s;

     location / {

         proxy_pass http://$host;

         proxy_set_header Host $host;
         
         proxy_set_header X-Forwarded-Proto $scheme;
     }
}
```

重启nginx即可生效

#### 代理测试

```
curl -v 'https://www.baidu.com' -x 127.0.0.1:8765
curl -v 'http://www.baidu.com' -x 127.0.0.1:8765
```

### chrome浏览器代理配置

chrome浏览器可以使用扩展来做域名的代理配置，此处推荐SwitchyOmega，可以到chrome应用商店下载安装该扩展

## 参考

[https://github.com/chobits/ngx\_http\_proxy\_connect\_module](https://github.com/chobits/ngx\_http\_proxy\_connect\_module)
