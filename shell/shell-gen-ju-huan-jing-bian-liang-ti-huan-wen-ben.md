# shell根据环境变量替换文本

## 背景

在实际工作中，有大量的配置文件需要配置，其中大部分内容都是相同的，然后每次配置的时候可能都是copy一个，然后修改其中的部分变量；这个是时候实际上可以根据每一类配置做一个模板，在新增配置的时候，通过模板+变量的方式生成新的配置文件

如果采用模板+变量的方式，那问题的难点就变成了如何替换模板中的变量；万幸的时候，shell中有个命令envsubstr可以解决此问题

## envsubst介绍

### 安装

enbsubst不是系统自带的命令，需要手动安装

mac环境下

```bash
brew install gettext
```

linux环境下不做介绍

### 使用

假设我们有一个模板a.tmpl，内容如下

```
hello $input
```

接下来，我们配置环境变量input

```
export input=world
```

接下来，我们来看下效果

```
envsubst < a.tmpl
```

效果如下

![](<../.gitbook/assets/image (2) (1).png>)

### 进阶

在docker中，可以搭配.env文件实现更灵活的配置变更，这里不做详细介绍

