---
description: prometheus清理数据
---

# prometheus清理数据

1、首先需要开启admin api

```bash
/path/to/prometheus --web.enable-admin-api ...
```

2、清理指标

```bash
curl -v -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=<metrics>{<condition>}&end=1716791083&start=1716787483'
```

3、清理

```bash
curl -v -X POST http://127.0.0.1:9090/api/v1/admin/tsdb/clean_tombstones
```
