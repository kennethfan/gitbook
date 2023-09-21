---
description: 因为墙的原因，pip的源在国内访问比较慢，因此希望pip安装的时候使用梯子来加速访问
---

# pip安装走socks代理

首先安装pysocks

```bash
pip install pysocks
```

使用socks代理安装

```bash
pip install <package> --proxy socks5://127.0.0.1:8234 # 请自行修改对应端口和ip
```

也是构建一个自动的shell命令

```bash
#!/bin/bash

echo "pip $@ --proxy socks5:127.0.0.1:8234"
pip $@ --proxy socks5://127.0.0.1:8234 # 请自行修改对应端口和ip
```
