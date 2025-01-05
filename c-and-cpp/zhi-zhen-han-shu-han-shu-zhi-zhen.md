# 指针函数&函数指针

## 引言

在看redis原理的时候，看到链表结构中，用到了函数指针，想起了之前容易弄混的指针函数和函数指针，所以记录一下

## 函数指针和指针函数的区别

### 定义

* 函数指针是指向函数地址的指针，本质是一个指针
* 指针函数是返回类型是指针的函数，本质上是函数

### 如何区分

看星号是否被括号包含：被括号包含是函数指针，反之是指针函数。

#### 函数指针示例

```c
#include <stdio.h>
// 申明一个函数，做加法操作
int sum(int a, int b) {
    return a + b;
}

// 申明一个函数，做乘法操作
int mul(int a, int b) {
    return a * b;
}

int main() {
    // 申明一个函数指针，参数两个int，返回int
    int (*method_point) (int a, int b);

    // 赋值给加法函数
    method_point = &sum;
    // 打印函数地址
    printf("%p\n", method_point);
    // 打印执行结果
    printf("%d\n", method_point(3, 5));

    // 赋值给乘法函数
    method_point = &mul;
    // 打印函数地址
    printf("%p\n", method_point);
    // 打印执行结果
    printf("%d\n", method_point(3, 5));

    return 0;
}
```

执行结果：

```shell
0x102256eb0
8
0x102256ed0
15
```

#### 指针函数示例

```c
#include <stdio.h>
// 定义一个函数，返回int指针类型
int *sum(int a, int b) {
    int c = a + b;
    return &c;
}

int main() {
    // 申明一个变量，赋值函数返回结果
    int* c = sum(3, 5);
    // 打印结果
  	printf("%d\n", *c);
  	return 0;
}
```
