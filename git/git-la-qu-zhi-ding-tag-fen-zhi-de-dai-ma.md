# git拉取指定tag/分支的代码

* 新建目录

```bash
mkdir workspace
```

* 初始化

```bash
cd workspace
git init .
git remote add origin git@github.com:kennethfan/docker-compose-tool.git
```

* 拉取分支/tag代码

```bash
git fetch --tags +refs/heads/*:refs/remotes/origin/*
commit=$(git rev-parse origin/${tag}^{commit}) # ${tag} 替换成对应域名
git checkout -f $commit
```
