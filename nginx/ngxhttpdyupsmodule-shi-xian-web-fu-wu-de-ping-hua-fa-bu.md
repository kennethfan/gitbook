# ngx\_http\_dyups\_module实现web服务的平滑发布

## 背景 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-bei-jing" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-bei-jing"></a>

公司有很多项目是对外的，对于服务的高可用有一定的要求。

因为项目一直在持续迭代，会经常发布更新，在发布过程中的高可用目前还是空白，需要解决

## 目标 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-mu-biao" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-mu-biao"></a>

解决项目发布过程中的高可用

## 方案分析 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-fang-an-fen-xi" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-fang-an-fen-xi"></a>

目前项目中的服务主要有两种

1.内部微服务，走nacos做服务发现和注册

2.类似网关服务，直接对接nginx

其中1类服务，通过nacos服务发现和注册已经解决了发布过程中的高可用问题；所以需要解决的是直接对接nginx的这一类服务

2类服务其部署架构如下

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

为了高可用，通常有多个节点，每个节点的功能相同，在发布过程中，其中某个节点会不可用。

因此发布过程中的思路几乎就是在发布过程中动态更新nginx upstream设置，其可选方案大致如下

| 方案一                      | 思路                                                                        | 优点                                                              | 缺点                    |
| ------------------------ | ------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------- |
| 手动处理                     | <p>发布前先手动修改nginx upstream并reload</p><p>发布完成之后再修改nginx upstream并reload</p> | 可靠                                                              | 需要人工介入，效率低，且人是不可靠的    |
| consul-template          | 通过consule自动监听服务，并通过consul-template动态修改ningx conf文件，reload                 | upstream变动自动完成，无需人工完成                                           | nginx需要频繁reload，有性能风险 |
| ngx\_http\_dyups\_module | ngx\_http\_dyups\_module 提供http接口，在发布过程中调用http接口更新upstream                | <p>1、upstream变动自动完成，无需人工完成</p><p>2、nginx无需reload</p><p><br></p> | <p><br></p>           |

综上评估，最终决定选用方案3，即ngx\_http\_dyups\_module方案

## 实施过程 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-shi-shi-guo-cheng" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-shi-shi-guo-cheng"></a>

### 安装过程 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-an-zhuang-guo-cheng" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-an-zhuang-guo-cheng"></a>

下载ngx\_http\_dyups\_module

```bash
cd /opt/data/soft/
git clone https://github.com/yzprofile/ngx_http_dyups_module.git
git pull --tag
git checkout v0.2.9
```

修改nginx源码

```bash
cd nginx-1.21.4 vim src/http/ngx_http_upstream.h
```

增加内容见下图

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

重新编译nginx

```bash
./configure --prefix=/opt/data/nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-ld-opt=-Wl,-rpath,/usr/local/lib --with-stream_ssl_module --with-stream_ssl_preread_module --add-module=/opt/data/soft/ngx_devel_kit --add-module=/opt/data/soft/set-misc-nginx-module --add-module=/opt/data/soft/lua-nginx-module --add-module=/opt/data/soft/ngx_http_dyups_module 
make
cp /opt/data/nginx/sbin/nginx /opt/data/nginx/sbin/nginx.bak
make install
```

## 验证 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-yan-zheng" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-yan-zheng"></a>

新增配置文件  /opt/data/nginx/conf/vhost-server/upstream.conf，内容如下

```bash
upstream t-plan-dev-gateway {
      server  192.168.2.89:8080;       
      server  192.168.2.89:8082;
}
```

新增配置文件 /opt/data/nginx/conf/vhost-server/ngx\_http\_dyups\_module.conf

```
server {        
    listen 10080; # 这个端口就是ngx_http_dyups_module作用端口，通过该端口做upstream更新；增加的端口需要添加防火墙配置，这里不做介绍        
    location / {                
        dyups_interface;        
    }
} 
# 测试upstream是否动态生效，生产环境可以删除
server {        
    server_name dev.dyups.com;        
    listen 80;         
    location / {                
        set $ups t-plan-dev-gateway; # 生产环境需要按照这种方式改造，upstream从写死变成nginx变量方式                
        proxy_pass http://$ups;        
    }
}
```

初始测试

```bash
curl -v http://127.0.0.1:10080/upstream/t-plan-dev-gateway
```

返回内容如下

```
* About to connect() to 127.0.0.1 port 10080 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 10080 (#0)
> GET /upstream/t-plan-dev-gateway HTTP/1.1
> User-Agent: curl/7.29.0> Host: 127.0.0.1:10080
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx
< Date: Wed, 08 Nov 2023 06:31:33 GMT
< Content-Length: 50
< Connection: keep-alive
<
server 192.168.2.89:8080
server 192.168.2.89:8082
* Connection #0 to host 127.0.0.1 left intact
```

可以看到返回了2个节点

接下来测试服务可用性

```bash
curl -v -H 'host: dev.dyups.com' 'http://127.0.0.1'
```

```
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: dev.dyups.com
>
< HTTP/1.1 404 Not Found
< Server: nginx
< Date: Wed, 08 Nov 2023 06:40:39 GMT
< Content-Type: application/json
< Content-Length: 130
< Connection: keep-alive
<
* Connection #0 to host 127.0.0.1 left intact
{"timestamp":"2023-11-08T06:40:39.920+00:00","path":"/","status":404,"error":"Not Found","message":null,"requestId":"52a1d925-12"}
```

可以看到服务正常

接下来验证删除upstream

```bash
curl -v -i -X DELETE http://127.0.0.1:10080/upstream/t-plan-dev-gateway
```

```
* About to connect() to 127.0.0.1 port 10080 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 10080 (#0)
> DELETE /upstream/t-plan-dev-gateway HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1:10080
> Accept: */*
>
< HTTP/1.1 200 OKHTTP/1.1 200 OK
< Server: nginxServer: nginx
< Date: Wed, 08 Nov 2023 07:03:53 GMTDate: Wed, 08 Nov 2023 07:03:53 GMT
< Content-Length: 7Content-Length: 7
< Connection: keep-aliveConnection: keep-alive 
<
* Connection #0 to host 127.0.0.1 left intact
success
```

看下upstream是否被删除

```bash
curl -v  http://127.0.0.1:10080/upstream/t-plan-dev-gateway
```

```
* About to connect() to 127.0.0.1 port 10080 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 10080 (#0)
> GET /upstream/t-plan-dev-gateway HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1:10080
> Accept: */*
>
< HTTP/1.1 404 Not Found
< Server: nginx
< Date: Wed, 08 Nov 2023 06:44:50 GMT
< Content-Length: 0
< Connection: keep-alive
<
* Connection #0 to host 127.0.0.1 left intact
```

404，表示upstream不存在

再看下服务可用

```bash
curl -v -H 'host: dev.dyups.com' 'http://127.0.0.1'
```

```
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: dev.dyups.com
>
< HTTP/1.1 502 Bad Gateway
< Server: nginx
< Date: Wed, 08 Nov 2023 07:04:34 GMT
< Content-Type: text/html; charset=utf-8< Content-Length: 150
< Connection: keep-alive
<
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx</center>
</body>
</html>* Connection #0 to host 127.0.0.1 left intact
```

返回502，所有没有可用的upstream

接下来，尝试更新upstream

```bash
curl -v  -d 'server 192.168.2.89:8082;'  http://127.0.0.1:10080/upstream/t-plan-dev-gateway
```

```
* About to connect() to 127.0.0.1 port 10080 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 10080 (#0)
> POST /upstream/t-plan-dev-gateway HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1:10080> Accept: */*
> Content-Length: 25
> Content-Type: application/x-www-form-urlencoded>
* upload completely sent off: 25 out of 25 bytes
< HTTP/1.1 200 OK
< Server: nginx
< Date: Wed, 08 Nov 2023 07:04:58 GMT
< Content-Length: 7
< Connection: keep-alive
<
* Connection #0 to host 127.0.0.1 left intact
success
```

再次查看upstream

```bash
curl -v http://127.0.0.1:10080/upstream/t-plan-dev-gateway
```

```
* About to connect() to 127.0.0.1 port 10080 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 10080 (#0)
> GET /upstream/t-plan-dev-gateway HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 127.0.0.1:10080
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx
< Date: Wed, 08 Nov 2023 07:05:21 GMT
< Content-Length: 25
< Connection: keep-alive
<
server 192.168.2.89:8082
* Connection #0 to host 127.0.0.1 left intact
```

已经有可用upstream，再次查看服务可用

```bash
curl -v -H 'host: dev.dyups.com' 'http://127.0.0.1'
```

```
* About to connect() to 127.0.0.1 port 80 (#0)
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: dev.dyups.com
>
< HTTP/1.1 404 Not Found
< Server: nginx
< Date: Wed, 08 Nov 2023 07:05:40 GMT
< Content-Type: application/json
< Content-Length: 130
< Connection: keep-alive
<
* Connection #0 to host 127.0.0.1 left intact
{"timestamp":"2023-11-08T07:05:40.192+00:00","path":"/","status":404,"error":"Not Found","message":null,"requestId":"79d33a64-16"}
```

服务恢复

证明方案可行

### 如何更新upstream <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-ru-he-geng-xin-upstream" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-ru-he-geng-xin-upstream"></a>

| 方案       | 思路                                                                        | 优点                        | 缺点                                                         |
| -------- | ------------------------------------------------------------------------- | ------------------------- | ---------------------------------------------------------- |
| jekins发布 | 更新jekins脚本，在服务重启前下掉节点，服务启动后加回节点                                           | 1、准确                      | <p>1、发布和部署架构耦合，配置繁琐</p><p>2、如何判断服务已经启动，需要服务改造，提供服务检测接口</p> |
| 应用自动完成   | <p>1、通过应用中添加listener，在服务启动后自动注册</p><p>2、通过添加shutdownHooker，在服务销毁时下掉节点</p> | <p>1、和发布解耦</p><p>2、自动</p> | 1、无法解决kill -9时，无法下掉节点的问题                                   |

因为线上正常流程是不允许kill -9的，出于解耦的目的，此处选择方案二

#### 应用改造 <a href="#ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-ying-yong-gai-zao" id="ngxhttpdyupsmodule-shi-xian-web-fu-wu-de-ping-hua-fa-bu-ying-yong-gai-zao"></a>

新增listener，监听应用启动和销毁事件

```java
@Slf4j
public class NginxRegistryListener implements ApplicationListener<ApplicationStartedEvent> {

    private NginxRegistryProp nginxRegistryProp;

    private int serverPort;

    public NginxRegistryListener(NginxRegistryProp nginxRegistryProp, int serverPort) {
        this.nginxRegistryProp = nginxRegistryProp;
        this.serverPort = serverPort;
    }

    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        log.info("NginxRegistryListener run");
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try {
                log.info("NginxRegistryListener.doUnregistry started");
                doUnregistry();
            } catch (Exception e) {
                log.error("NginxRegistryListener.doUnregistry error. ", e);
            }
        }));

        try {
            log.info("NginxRegistryListener.doRegistry started");
            doRegistry();
        } catch (Exception e) {
            log.error("NginxRegistryListener.doRegistry error. ", e);
        }
    }

    private List<String> getUpstreamList() throws IOException, URISyntaxException {
        // 创建Httpclient对象
        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            // 创建uri
            URIBuilder builder = new URIBuilder(this.nginxRegistryProp.getRegistryUrl());
            URI uri = builder.build();

            // 创建http GET请求
            HttpGet httpGet = new HttpGet(uri);
            // 执行请求
            try (CloseableHttpResponse response = httpclient.execute(httpGet)) {
                // 判断返回状态是否为200
                if (response.getStatusLine().getStatusCode() == HttpStatus.SC_NOT_FOUND) {
                    return new ArrayList<>();
                }

                if (response.getStatusLine().getStatusCode() != HttpStatus.SC_OK) {
                    throw new IOException("http request error, url= " + this.nginxRegistryProp.getRegistryUrl());
                }

                return IOUtils.readLines(response.getEntity().getContent(), StandardCharsets.UTF_8);
            }
        }
    }

    private void doRegistry() throws IOException, URISyntaxException {
        if (nginxRegistryProp == null
                || StringUtils.isBlank(nginxRegistryProp.getRegistryUrl())
                || StringUtils.isBlank(nginxRegistryProp.getServiceIp())) {
            log.info("doRegistry, nothing todo, nginxRegistryProp={}", nginxRegistryProp);
            return;
        }
        List<String> upstreamList = getUpstreamList();
        String destUpstream = this.buildUpstream();

        if (upstreamList.contains(destUpstream)) {
            log.info("doRegistry, upstream registered already, nginxRegistryProp={}", nginxRegistryProp);
            return;
        }

        upstreamList.add(destUpstream);
        updateUpstream(upstreamList);
    }

    private void doUnregistry() throws IOException, URISyntaxException {
        if (nginxRegistryProp == null
                || StringUtils.isBlank(nginxRegistryProp.getRegistryUrl())
                || StringUtils.isBlank(nginxRegistryProp.getServiceIp())) {
            log.info("doUnregistry, nothing todo, nginxRegistryProp={}", nginxRegistryProp);
            return;
        }
        List<String> upstreamList = getUpstreamList();
        String destUpstream = this.buildUpstream();

        if (!upstreamList.remove(destUpstream)) {
            log.info("doUnregistry, upstream unregistered already, nginxRegistryProp={}", nginxRegistryProp);
            return;
        }

        if (CollectionUtils.isEmpty(upstreamList)) {
            deleteUpstream();
            return;
        }

        updateUpstream(upstreamList);
    }

    private String buildUpstream() {
        return String.format("server %s:%d", nginxRegistryProp.getServiceIp(), serverPort);
    }

    private void updateUpstream(List<String> upstreamList) throws IOException, URISyntaxException {
        StringBuilder sb = new StringBuilder();
        for (String upstream : upstreamList) {
            sb.append(upstream)
                    .append(";");
        }

        String postStr = sb.toString();

        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            // 创建uri
            URIBuilder builder = new URIBuilder(this.nginxRegistryProp.getRegistryUrl());
            URI uri = builder.build();

            // 创建http POST请求
            HttpPost httpPost = new HttpPost(uri);
            httpPost.addHeader("Content-Type", "application/x-www-form-urlencoded");
            httpPost.setEntity(new StringEntity(postStr, StandardCharsets.UTF_8));
            // 执行请求
            try (CloseableHttpResponse response = httpclient.execute(httpPost)) {
                // 判断返回状态是否为200

                if (response.getStatusLine().getStatusCode() != HttpStatus.SC_OK) {
                    throw new IOException("http request error, url= " + this.nginxRegistryProp.getRegistryUrl());
                }

                String content = IOUtils.toString(response.getEntity().getContent(), StandardCharsets.UTF_8);
                if (!StringUtils.equalsIgnoreCase("success", content)) {
                    throw new IOException("http request error, url= " + this.nginxRegistryProp.getRegistryUrl());
                }
            }
        }
    }

    private void deleteUpstream() throws IOException, URISyntaxException {
        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            // 创建uri
            URIBuilder builder = new URIBuilder(this.nginxRegistryProp.getRegistryUrl());
            URI uri = builder.build();

            // 创建http DELETE请求
            HttpDelete httpDelete = new HttpDelete(uri);
            // 执行请求
            try (CloseableHttpResponse response = httpclient.execute(httpDelete)) {
                // 判断返回状态是否为200

                if (response.getStatusLine().getStatusCode() != HttpStatus.SC_OK) {
                    throw new IOException("http request error, url= " + this.nginxRegistryProp.getRegistryUrl());
                }

                String content = IOUtils.toString(response.getEntity().getContent(), StandardCharsets.UTF_8);
                if (!StringUtils.equalsIgnoreCase("success", content)) {
                    throw new IOException("http request error, url= " + this.nginxRegistryProp.getRegistryUrl());
                }
            }
        }
    }

}

```

新增配置类

```java
@Setter
@Getter
@RefreshScope
@ConfigurationProperties(prefix = "nginx.registry")
public class NginxRegistryProp {
    /**
     * nginx 注册地址
     */
    private String registryUrl;

    /**
     * 服务ip
     */
    private String serviceIp;

    @Override
    public String toString() {
        return "NginxRegistryProp{" +
                "registryUrl='" + registryUrl + '\'' +
                ", serviceIp='" + serviceIp + '\'' +
                '}';
    }
}
```

配置新增如下

```yaml
nginx:  
  registry:    
    registry-url: http://192.168.2.77:10080/upstream/t-plan-dev-gateway  # nginx upstream 变更地址    
    service-ip: 192.168.25.86 # 服务节点ip
```

服务启动过程日志如下

```
2023-11-08 17:07:03.149  INFO 31717 --- [           main] o.a.coyote.http11.Http11NioProtocol      : Starting ProtocolHandler ["http-nio-8421"]
2023-11-08 17:07:03.182  INFO 31717 --- [           main] c.dreamkey.chain.brain.BrainApplication  : Started BrainApplication in 3.901 seconds (JVM running for 9.654)
2023-11-08 17:07:03.183  INFO 31717 --- [           main] d.f.u.lister.NginxRegistryListener       : NginxRegistryListener run
2023-11-08 17:07:03.184  INFO 31717 --- [           main] d.f.u.lister.NginxRegistryListener       : NginxRegistryListener.doRegistry started
2023-11-08 17:07:03.421  INFO 31717 --- [           main] c.a.c.n.refresh.NacosContextRefresher    : listening config: dataId=coupon-chain-brain.yml, group=LQT
2023-11-08 17:07:03.421  INFO 31717 --- [           main] c.dreamkey.chain.brain.BrainApplication  : ......链券通服务启动成功!
```

可以看到启动后有调用NginxRegistryListener.doRegistry注册服务

看下upstream情况

```bash
curl -v 'http://192.168.2.77:10080/upstream/t-plan-dev-gateway'
```

```
*   Trying 192.168.2.77:10080...
* Connected to 192.168.2.77 (192.168.2.77) port 10080 (#0)
> GET /upstream/t-plan-dev-gateway HTTP/1.1
> Host: 192.168.2.77:10080
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx
< Date: Wed, 08 Nov 2023 09:07:27 GMT
< Content-Length: 51
< Connection: keep-alive
<
server 192.168.2.89:8080
server 192.168.25.86:8421
* Connection #0 to host 192.168.2.77 left intact
```

可以看到有一个端口8421的节点，表示服务启动后注册成功

接下来看销毁，日志如下

```
2023-11-08 17:07:42.110  WARN 31717 --- [       Thread-1] c.a.n.common.http.HttpClientBeanHolder   : [HttpClientBeanHolder] Start destroying common HttpClient
2023-11-08 17:07:42.111  INFO 31717 --- [       Thread-7] d.f.u.lister.NginxRegistryListener       : NginxRegistryListener.doUnregistry started
2023-11-08 17:07:42.113  WARN 31717 --- [       Thread-1] c.a.n.common.http.HttpClientBeanHolder   : [HttpClientBeanHolder] Destruction of the end
2023-11-08 17:07:42.137  INFO 31717 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2023-11-08 17:07:42.145  INFO 31717 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
```

可以看到服务销毁前有调用NginxRegistryListener.doUnregistry下掉节点

再看下upstream节点情况

```bash
curl -v 'http://192.168.2.77:10080/upstream/t-plan-dev-gateway'
```

```
*   Trying 192.168.2.77:10080...
* Connected to 192.168.2.77 (192.168.2.77) port 10080 (#0)
> GET /upstream/t-plan-dev-gateway HTTP/1.1
> Host: 192.168.2.77:10080
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx
< Date: Wed, 08 Nov 2023 09:07:57 GMT
< Content-Length: 25
< Connection: keep-alive
<server 192.168.2.89:8080
* Connection #0 to host 192.168.2.77 left intact
```

可以看到8421节点已经被摘除，证明方案可行

## 参考

[https://github.com/yzprofile/ngx\_http\_dyups\_module](https://github.com/yzprofile/ngx_http_dyups_module)
