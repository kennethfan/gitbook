# JNR调用本地动态链接库

首先准备动态链接库

准备一段C代码

```c
int add(int a, int b)
    return a + b;
}
```

编译成动态链接库

```bash
gcc -shared -o add.so add.c
```

复制add.so到java项目resource目录

引入jnr依赖

```markup
<dependency>
    <groupId>com.github.jnr</groupId>
    <artifactId>jnr-ffi</artifactId>
    <version>2.1.10</version>
</dependency>
```

测试代码

```java
import jnr.ffi.LibraryLoader;

public class JNRTest {
    public interface NativeLib {
        NativeLib INSTANCE = LibraryLoader.create(NativeLib.class).load("add.so");

        int add(int a, int b);
    }

    public static void main(String[] args) {
        NativeLib instance = NativeLib.INSTANCE;

        System.out.println(instance.add(1, 3));
    }
}
```

运行结果如下

![](<../.gitbook/assets/image (1) (1).png>)
