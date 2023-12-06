---
description: java使用page cache
---

# java使用page cache

文件操作使用page cache可以减少io次数

<pre class="language-java"><code class="lang-java">import java.io.*;

public class PageCacheMain {

    public static void main(String[] args) throws IOException {
        long begin, end;

        int[] sizes = {32, 64, 128, 256, 512, 1024};

        File file;
        for (int size : sizes) {
            file = new File("data2.txt");
            if (file.exists()) {
                file.delete();
            }
            begin = System.currentTimeMillis();
            stream(size);
            end = System.currentTimeMillis();
            System.out.println("stream perSize=" + size + " cost=" + (end - begin));

            file = new File("data4.txt");
            if (file.exists()) {
                file.delete();
            }
            begin = System.currentTimeMillis();
            pageCache(size);
            end = System.currentTimeMillis();
            System.out.println("pageCache perSize=" + size + " cost=" + (end - begin));
        }
    }

    private static void stream(int perSize) throws IOException {
        byte[] bytes = new byte[perSize];
        try (FileInputStream inputStream = new FileInputStream("data1.txt"); FileOutputStream outputStream = new FileOutputStream("data2.txt")) {
            while (inputStream.available() > 0) {
                inputStream.read(bytes);
                outputStream.write(bytes);
            }
        }
    }
<strong>
</strong>    private static void pageCache(int perSize) throws IOException {
        byte[] bytes = new byte[perSize];
        try (BufferedInputStream inputStream = new BufferedInputStream(new FileInputStream("data3.txt")); BufferedOutputStream outputStream = new BufferedOutputStream(new FileOutputStream("data4.txt"))) {
            while (inputStream.available() > 0) {
                inputStream.read(bytes);
                outputStream.write(bytes);
            }
        }
    }
<strong>}
</strong></code></pre>

日志如下

```
stream perSize=32 cost=202554
pageCache perSize=32 cost=44176
stream perSize=64 cost=100262
pageCache perSize=64 cost=23576
stream perSize=128 cost=49865
pageCache perSize=128 cost=12733
stream perSize=256 cost=25347
pageCache perSize=256 cost=7940
stream perSize=512 cost=13816
pageCache perSize=512 cost=4822
stream perSize=1024 cost=7683
pageCache perSize=1024 cost=3698
```
