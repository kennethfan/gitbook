---
description: prometheus抓取数据配置
---

# prometheus抓取数据配置

## 背景

最近在使用prometheus做监控，其中有不少服务和组件都没法直接暴露端口供prometheus访问，因此各种prometheus exporter通过nginx转发，同时访问限制了白名单，所以涉及到prometheus一些复杂配置，这里记录下

```yaml
scrape_configs:
  - job_name: "mysql-master" # 任务名称
    metrics_path: '/metrics' # 指标所在路径，此处是nginx分发路径
    params: # url参数
        path: ['/metrics'] 
        up: ['192.168.10.197:9104']
    scheme: 'https' # 访问协议，http的话可以不用
    proxy_url: 'http://192.168.3.170:8765' # 代理地址，没有使用可以不用
    static_configs:
      - targets: ["parking-center.co-coupon.com:443"] # nginx代理地址
        labels:
          instance: 'parking-mysql-master' # 额外标签，此处使用instance覆盖掉默认的，默认会从targets取
```
