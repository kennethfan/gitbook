# zsh-字符串常用操作

zsh 的字符串处理功能，要比绝大多数编程语言自带的字符串函数库或者类库要强大（在不依赖外部命令的情况）。同时各种用法也比较怪异，很多时候简洁性和可读性是有矛盾的，很难兼顾。而 shell 的使用场景决定简洁性是不能被牺牲掉的，即使用 Python 这样比较简洁的语言来处理字符串，很多时候也只能写出冗长的代码，而 zsh 经常可以一行搞定（可能有人想到了 Perl，Perl 在处理文本方面确实有比较明显的优势，但使用 Perl 的话也要承担更多的成本），如果再加上适当地使用外部命令，基本可以应付大多数字符串处理场景。因为字符串处理的内容比较丰富，我会分多篇文章写。本篇只涉及最基础和常用的字符串操作，包括字符串的拼接、切片、截断、查找、遍历、替换、匹配、大小写转换、分隔等等。

## 字符串长度

```sh
% str=abcde
% echo $#str
5

# 读取函数或者脚本的第一个参数的长度
% echo $#1
```

## 字符串拼接

```sh
% str1=abc
% str2=def

% str2+=$str1
% echo $str2
defabc

% str3=$str1$str2
abcdefabc
```

## 字符串切片

```sh
% str=abcdef
% echo $str[2,4]
bcd
% echo $str[2,-1]
bcdef

# $1 是文件或者函数的第一个参数
echo ${1[2,4]}
```

字符串切片还有另一种风格的方法，即 bash 风格，功能大同小异。通常没有必要用这个，而且因为字符位置是从 0 开始算，容易混淆。

```sh
% str=abcdef
% echo ${str:1:3}
bcd
% echo ${str:1:-1}
bcde
```

### 字符串截断

```sh
% str=abcdeabcde

# 删除左端匹配到的内容，最小匹配
% echo ${str#*b}
cdeabcde

# 删除右端匹配到的内容，最小匹配
% echo ${str%d*}
abcdeabc

# 删除左端匹配到的内容，最大匹配
% echo ${str##*b}
cde

# 删除右端匹配到的内容
% echo ${str%%d*}
abc
```

## 字符串查找

子字符串定位。

```sh
% str=abcdef

# 这里用的是 i 的大写，不是 L 的小写
% echo $str[(I)cd]
3

# I 是从右往左找，如果找不到则为 0, 方便用来判断
% (($str[(I)cd])) && echo good
good

# 找不到则为 0
% echo $str[(I)cdd]
0

# 也可以使用小 i，小 i 是从左往右找，找不到则返回数组大小 + 1
% echo $str[(i)cd]
3

% echo $str[(i)cdd]
7
```

## 遍历字符

```sh
% str=abcd

% for i ({1..$#str}) {
>    echo $str[i]
>}
a
b
c
d
```

## 字符串替换

按内容替换和删除字符。

```sh
% str=abcdefg

# 一对一地替换
% str[2]=1
% echo $str
a1cdefg

# 可以多对多（也包括一对多和多对一）地替换字符，两边的字符数量不需要一致。
# 把第二、三个字符替换成 2345
% str[2,3]=2345
% echo $str
a2345defg
```

## 判断字符串变量是否存在

如果用 `[[ "$strxx" == "" ]]` ，那无法区分变量是没有定义还是内容为空，在某些情况是需要区分二者的。

```sh
% (($+strxx)) && echo good

% strxx=""
% (($+strxx)) && echo good
good
```

`(($+var))` 的用法也可以用来判断其他类型的变量，如果变量存在则返回真（0），否则返回假（1）。

## 字符串匹配判断

### 判断是否包含字符串

```sh
% str1=abcd
% str2=bc

% [[ $str1 == *$str2* ]] && echo good
good
```

### 正则表达式匹配

```sh
% str=abc55def

# 少量字符串的话，尽量不要用 grep
# 本文不讲正则表达式格式相关内容
# 另外 zsh 有专门的正则表达式模块
% [[ $str =~ "c[0-9]{2}\de" ]] && echo a
a
```

## 大小写转换

```sh
% str="ABCDE abcde"

# 转成大写，(U) 和 :u 两种用法效果一样
% echo ${(U)str} --- ${str:u}
ABCDE ABCDE --- ABCDE ABCDE

# 转成小写，(L) 和 :l 两种用法效果一样
% echo ${(L)str} --- ${str:l}
abcde abcde --- abcde abcde

# 转成首字母大写
% echo ${(C)str} 
Abcde Abcde
```

## 目录文件名截取

```sh
% filepath=/a/b/c.x

# :h 是取目录名，即最后一个 / 之前的部分，如果没有 / 则为 .
% echo ${filepath:h}
/a/b

# :t 是取文件名，即最后一个 / 之后的部分，如果没有 / 则为字符串本身
% echo ${filepath:t}
c.x

# :e 是取文件扩展名，即文件名中最后一个点之后的部分，如果没有点则为空
% echo ${filepath:e}
x

# :r 是去掉末尾扩展名的路径
% echo ${filepath:r}
/a/b/c
```

## 字符串分隔

```sh
# 使用空格作为分隔符，多个空格也只算一个分隔符
% str='aa bb cc dd'
% echo ${str[(w)2]}
bb
% echo ${str[(w)3]}
cc

# 指定分隔符
% str='aa--bb--cc'
# 如果分隔符是 : 就用别的字符作为左右界，比如 ws.:.
% echo ${str[(ws:--:)3]}
cc
```

## 多行字符串

```sh
% str="line1
> line2"
% echo $str
line1
line2
```

## 读取文件内容到字符串

```sh
# 比用 str=$(cat filename) 性能好很多
str=$(<filename)

# 比用 cat filename 性能好很多，引号不能省略，省略了之后就不换行了
echo "$(<filename)"

# 遍历每行，引号不能省略
for i (${(f)"$(<filename)"}) {
    echo $i
}
```

### 读取文件指定行

文件 test.txt 内容如下：

```
line 1. apple
line 2. orange
```

```sh
# 小文件或者需要频繁调用时，尽量不要用 sed
% echo ${"$(<test.txt)"[(f)2]}
line 2. orange

# 输出包含 “ang” 的第一行
% echo ${"$(<test.txt)"[(fr)*ang*]}
line 2. orange

# 输出包含 pp 的第一行，但从左截掉 “line” 4个字符。
echo ${"$(<test.txt)"[(fr)*pp*]#line}
```

## 读取进程输出到字符串

读进程输出和读文件类似。

上边字符串相关的处理，直接把 `$(<test.txt)` 换成 `$(命令)` 即可。如果一定需要一个文件名，可以这样。

```sh
# 返回 fd 路径，优先使用，但某些场景会出错
% wc -l <(ps)
4 /proc/self/fd/11

# 临时文件，会自动删除，适合上边用法出错的情况
% wc -l =(ps)
3 /tmp/zshMWDpqD
```

## 参考

[zsh开发指南](https://github.com/goreliu/zshguide/)
