---
description: 过期jar删除
---

# 过期jar删除

```bash
# 过期jar删除
0 1 * * 1  ls /path/to/*.jar | while read f; do [ $(lsof $f |wc -l) -eq 0 ] && rm -f $f; done
```
