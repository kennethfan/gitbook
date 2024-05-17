---
description: git设置用户信息缓存
---

# git设置用户信息缓存

## 背景

在我们使用http协议和git仓库交互时，每次push/pull都需要手动输入一遍用户名和密码非常麻烦



通过配置缓存的方式，我们可以在一段时间内免去输入用户和密码的烦恼

```bash
config --global credential.helper 'cache --timeout=86400' # 时间单位是秒，这里配置的一天
```
