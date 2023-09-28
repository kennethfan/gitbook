# javac命令编译\&java命令运行代码

## 背景

项目中有很多诡异的问题，经常会猜测某种可能，但是又不敢确定；此时需要一些临时代码来测试，如果直接写到项目里面，会比较麻烦

此文记录下写临时测试代码到一个单独文件并运行的过程

### 编辑代码

```bash
vim XxxxTest.java # vim 编辑，内容随意
```

### 编译代码

```bash
javac -cp <依赖jar路径1>:<依赖jar路径2> XxxxTest.java
```

运行代码

```bash
javac -cp <依赖jar路径1>:<依赖jar路径2> XxxxTest
```
