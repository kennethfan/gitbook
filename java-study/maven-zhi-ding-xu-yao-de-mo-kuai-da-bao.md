---
description: maven指定需要的模块打包
---

# maven指定需要的模块打包

```bash
mvn clean install -pl module1,module2 -am # -am参数表示依赖模块一同编译
```
