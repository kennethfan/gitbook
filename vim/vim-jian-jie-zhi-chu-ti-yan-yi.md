# Vim简介之初体验(一)

**Vim**是从[vi](https://zh.wikipedia.org/wiki/Vi)发展出来的一个[文本编辑器](https://zh.wikipedia.org/wiki/%E6%96%87%E6%9C%AC%E7%BC%96%E8%BE%91%E5%99%A8)。代码补完、编译及错误跳转等方便编程的功能特别丰富，在程序员中被广泛使用。和[Emacs](https://zh.wikipedia.org/wiki/Emacs)并列成为[类Unix系统](https://zh.wikipedia.org/wiki/%E7%B1%BBUnix%E7%B3%BB%E7%BB%9F)用户最喜欢的编辑器。——[维基百科](https://zh.wikipedia.org/wiki/Vim)

## Vim和Emacs之争

**Vim是编辑器之神，Emacs是神的编辑器。**

\[为何Emacs和Vim被称为两大神器]\(为何 Emacs 和 Vim 被称为两大神器)

## Vim常见的模式

### 普通模式

在普通模式中，用的编辑器命令，比如移动光标，删除文本等等。这也是Vim启动后的默认模式。

### 插入模式

在这个模式中，大多数按键都会向文本[缓冲](https://zh.wikipedia.org/w/index.php?title=%E7%BC%93%E5%86%B2\&action=edit\&redlink=1)中插入文本。大多数新用户希望文本编辑器编辑过程中一直保持这个模式。在插入模式中，按Esc键可以回到普通模式。

### 命令行模式

在命令行模式中可以输入会被解释成并执行的文本。例如执行命令（":"键），搜索（"/"和"?"键）或者过滤命令（"!"键）。在命令执行之后，Vim返回到命令行模式之前的模式，通常是普通模式。
