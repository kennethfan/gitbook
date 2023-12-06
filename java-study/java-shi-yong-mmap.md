---
description: java使用mmap
---

# java使用mmap

文件操作，可以通过mmap将文件映射到内存，提升写入性能

```java
public class MmapMain {

    private static final int FILE_SIZE = 1 << 22;

    public static void main(String[] args) throws IOException {
        long begin = System.currentTimeMillis();
        random();
        long end = System.currentTimeMillis();
        System.out.println("random cost=" + (end - begin));

        begin = System.currentTimeMillis();
        stream();
        end = System.currentTimeMillis();
        System.out.println("stream cost=" + (end - begin));

        begin = System.currentTimeMillis();
        mmap();
        end = System.currentTimeMillis();
        System.out.println("mmap cost=" + (end - begin));
    }


    private static void random() throws IOException {
        RandomAccessFile raf = new RandomAccessFile("random.txt", "rw");
        byte[] bytes = new byte[1];
        bytes[0] = (byte) 'a';
        for (int i = 0; i < FILE_SIZE; i++) {
            raf.write(bytes);
        }
    }

    private static void stream() throws IOException {
        byte[] bytes = new byte[1];
        try (FileOutputStream outputStream = new FileOutputStream("stream.txt")) {
            bytes[0] = (byte) 'a';
            for (int i = 0; i < FILE_SIZE; i++) {
                outputStream.write(bytes);
            }
        }
    }

    private static void mmap() throws IOException {
        RandomAccessFile raf = new RandomAccessFile("mmap.txt", "rw");
        MappedByteBuffer mbb = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, FILE_SIZE);
        byte bt = (byte) 'a';
        for (int i = 0; i < FILE_SIZE; i++) {
            mbb.put(bt);
        }
    }
}
```

执行日志如下

```
normal cost=15097
mmap cost=19
```

可以看出，在64M文件顺序写入的情况下，mmap性能比random模式搞出好几个数量级
