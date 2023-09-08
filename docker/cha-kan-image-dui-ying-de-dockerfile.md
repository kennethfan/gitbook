# 查看image对应的dockerfile

```bash
docker history --format {{.CreatedBy}} --no-trunc=true <镜像id> | sed '1!G;h;$!d'
```

也可以写个shell脚本

```bash
#!/bin/bash


if [ $# -lt 1 ]; then
	echo "Usage $0 <image>"
	exit
fi


docker history --format {{.CreatedBy}} --no-trunc=true $1 | sed '1!G;h;$!d'
```
