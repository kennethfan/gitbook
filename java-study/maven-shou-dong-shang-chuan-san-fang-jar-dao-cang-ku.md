---
description: maven手动上传三方jar到仓库
---

# maven手动上传三方jar到仓库

## 背景

在和外部对接的过程中，合作伙伴通常会给一些sdk，但是sdk又不会放到公共仓库，因此需要手动上传maven私服管理

操作如下

```bash
mvn deploy:deploy-file -Dfile=<文件路径> -DgroupId=<groupId> -DartifactId=<artifactId> -Dversion=<version> -Dpackaging=jar -DrepositoryId=<repositoryId>
```

上传之前需要手动配置下插件

```markup
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-deploy-plugin</artifactId>
    <version>2.7</version>
    <configuration>
        <url>仓库地址</url>
    </configuration>
</plugin>
```
