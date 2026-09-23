# IO 流

## 一句话理解

> **输入流（读）** → 数据流进程序
> **输出流（写）** → 数据流出程序

流就像一根水管，一头连程序，一头连文件/网络/键盘。

## 两条主线

| 种类 | 用途 | 常见类 |
|------|------|--------|
| **字节流** | 读写任何文件（图片、视频、文本等） | `FileInputStream` / `FileOutputStream` |
| **字符流** | 专门读写文本文件（.txt、.java） | `FileReader` / `FileWriter` |

## 代码示例

```java
// 读文本文件
try (FileReader fr = new FileReader("test.txt")) {
    int ch;
    while ((ch = fr.read()) != -1) {
        System.out.print((char) ch);
    }
} // try-with-resources 自动关流，不用手写 finally
```
