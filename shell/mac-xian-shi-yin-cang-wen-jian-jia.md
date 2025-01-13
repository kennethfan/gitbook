---
description: mac显示隐藏文件夹
---

# mac显示隐藏文件夹

mac系统默认会将.开头的文件或者文件夹隐藏，因此使用Finder(访达)时默认是不是显示这些的

如果是单次展示这隐藏文件，可以使用快捷键`Command` + `Shift` + `.`

`如果想永久更改配置，可以如下操作，打开终端`

更新配置

```bash
defaults write com.apple.Finder AppleShowAllFiles -bool true
```

关闭所有Finder(访达)进程

```bash
killall Finder
```

重新打开Finder(访达)进程就会生效

\
