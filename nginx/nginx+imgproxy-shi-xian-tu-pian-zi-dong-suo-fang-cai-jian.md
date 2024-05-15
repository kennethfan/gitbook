---
description: nginx+imgproxy实现图片自动缩放裁剪
---

# nginx+imgproxy实现图片自动缩放裁剪

## 背景

在项目开发过程中，经常会遇到一张图片在不同场景下需要使用不同的尺寸，本文旨在使用nginx+imgproxy实现图片的自动裁剪

## 软件要求

* git，用于拉去nginx扩展代码
* docker，用于拉起imgproxy
* nginx，略
* imgproxy，实现图片自动裁剪，以及转码；本文使用docker拉起，不做安装
* lua，实现文件名的特殊处理
* ngx\_devel\_kit，nginx模块，用于安装lua-nginx-module和set-misc-nginx-module
* lua-nginx-module nginx扩展，实现nginx调用lua
* set-misc-nginx-module针对uri做encode/decode

### 软件安装

### docker

```bash
sudo yum -y update && sudo yum -y install docker # 安装
sudo systemctl enable docker && sudo systemctl start docker
```

### git

```bash
sudo yum -y install git
```

### imgproxy

imgproxy拉起

```bash
sudo docker pull darthsim/imgproxy:latest # imgproxy镜像拉取
sudo docker run -d --name imgproxy --restart always -p 8081:8080 -v /opt/data/backend/file:/app/images --env="IMGPROXY_LOCAL_FILESYSTEM_ROOT=/app/images" --user root --privileged darthsim/imgproxy
```

\--user root 根据实际情况，如果文件默认都有读权限，可以不需要

\--env="IMGPROXY\_LOCAL\_FILESYSTEM\_ROOT=/app/images"，用于指定imgproxy图片根目录

\-v /opt/data/backend/file:/app/images 用于映射本地目录到imgproxy图片根目录

### lua

```bash
cd /opt/data/soft
wget http://luajit.org/download/LuaJIT-2.0.5.tar.gz
tar -zvxf LuaJIT-2.0.5.tar.gz
cd LuaJIT-2.0.5
sudo make && sudo make install
  
sudo echo export LUAJIT_LIB=/usr/local/lib >> /etc/profile
sudo echo export LUAJIT_INC=/usr/local/include/luajit-2.0 >> /etc/profile  
```

### ngx\_devel\_kit

```bash
git clone https://github.com/vision5/ngx_devel_kit.git
cd ngx_devel_kit
git checkout v0.3.0
```

### lua-ngix-module

```bash
cd /opt/data/soft
git clone https://github.com/openresty/lua-nginx-module.git
cd lua-nginx-module
git checkout v0.10.11
```

### set-misc-nginx-module

```bash
cd /opt/data/soft
git clone https://github.com/openresty/set-misc-nginx-module.git
cd set-misc-nginx-module
git checkout v0.33
```

### nginx 重新编译/安装

如果重新编译，可以看下nginx之前编译参数

```bash
/path/to/nginx -V
```

```
cd /path/to/nginx_src/
sudo ./configure ${历史参数} --with-ld-opt=-Wl,-rpath,/usr/local/lib --add-module=/opt/data/soft/ngx_devel_kit --add-module=/opt/data/soft/set-misc-nginx-module --add-module=/opt/data/soft/lua-nginx-module
sudo make -j2 # 编译
sudo cp /path/to/nginx /path/to/nginx.bak #备份nginx二进制文件
sudo make install
```

#### nginx配置

开启nginx缓存

```
http {
  proxy_cache_path /opt/data/nginx/cache levels=1:2 keys_zone=imgproxy:300m max_size=5g;
 
  # 代理缓存配置说明
  # /opt/data/nginx/cache 缓存路径
  # levels=1:2 缓存目录的层级结构
  # keys_zone=imgproxy:300m 缓存名字为imgproxy，缓冲区大小为300m
  # max_size=5g 缓存的最大容量为 5GB
}
```

图片自动裁剪/转码

```
server {
    # 监听端口
    listen 80;
    # 域名
    server_name domain.com; # 图片访问的域名
 
    # 这个路径是基于框架的统一文件访问前缀
    location /api/static/file {  # 这里/api/static/file是图片访问的基础路径
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;  
               
        set $encoded_filename "";
        if ($request_uri ~* "^/api/static/file/(.*)\?.*$") {
            # filename解码,避免重复URL编码
            set_unescape_uri $decoded_filename $1;
            # 替换filename中的@为%40,否则imgproxy会认为@是分隔符
            set_by_lua_block $encoded_filename {
                local filename = ngx.var.decoded_filename
                local encoded = string.gsub(filename, "@", "%%40")
                return encoded
            }
        }        
        # 根据路径参数p=true来判断是否进行转发
        if ($arg_p = 'true') {
           # 将图片处理请求根据imgproxy的规则进行路径重写
           rewrite ^(.*)$ /signature/resize:$arg_f:$arg_w:$arg_h:0/plain/local:///$encoded_filename@$arg_t break;
           # 转发给imgproxy进行处理
           proxy_pass http://127.0.0.1:8081;
           expires max;
           # 基于浏览器的缓存 携带此请求头浏览器会缓存图片
           add_header Cache-Control "public, no-transform";
        }
        # 没有携带参数p则直接请求原图 10001为后端服务的路径
        proxy_pass http://127.0.0.1:10001;
        # 启用基于nginx的代理缓存 指定缓存名称
        proxy_cache imgproxy;
        # 配置缓存key的规则
        proxy_cache_key $scheme$proxy_host$uri$is_args$args;
        # 基于http状态码来配置缓存的过期时间 这里设置为30天
        proxy_cache_valid  200 304 302 30d;
        expires max;
        add_header Cache-Control "public, no-transform";
    }
}
```

nginx重启

```bash
sudo /path/to/nginx -t # 校验配置文件
sudo /path/to/nginx -s reload # 重启nginx
```

参数说明

```
p：true则代理图片

w：图片宽度

h：图片高度

f：缩放选项 fit保持宽高比 fill不保持宽高比直接裁剪

t：图片类型 建议jpg 有特殊要求的可使用png
```

比如原图

http://www.domain.com/api/static/file/xxxxxx.png

缩略图

http://www.domain.com/api/static/file/xxxxxx.png?p=true\&w=300\&t=jpg\&f=fit\
