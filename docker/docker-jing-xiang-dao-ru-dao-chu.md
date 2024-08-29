---
description: docker镜像导入导出
---

# docker镜像导入导出

镜像导出

```bash
docker save -o image.tar.gz image:tag
```

镜像导入

```bash
docker load -i image.tar.gz
```
