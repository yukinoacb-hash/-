# IO 流

## IO 流分类

```
IO流
 ├── 字节流（读写任意文件）
 │    ├── FileInputStream    — 读
 │    └── FileOutputStream   — 写
 ├── 字符流（读写纯文本）
 │    ├── FileReader         — 读
 │    └── FileWriter         — 写
 ├── 缓冲流（提高效率）
 │    ├── BufferedInputStream / BufferedOutputStream
 │    ├── BufferedReader     — readLine() 按行读取
 │    └── BufferedWriter
 ├── 转换流（指定编码）
 │    ├── InputStreamReader  — 字节→字符，可指定编码
 │    └── OutputStreamWriter — 字符→字节
 ├── 对象流（读写整个对象）
 │    ├── ObjectInputStream  — readObject()
 │    └── ObjectOutputStream — writeObject()
 └── 打印流（方便打印，不抛异常）
      ├── PrintStream        — System.out 就是它
      └── PrintWriter        — 字符版
```

## 文件拷贝（核心范例）

```java
try (FileInputStream fis = new FileInputStream("D:\\src.jpg");
     FileOutputStream fos = new FileOutputStream("D:\\dest.jpg")) {
    byte[] buffer = new byte[1024];
    int len;
    while ((len = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, len);
    }
} catch (IOException e) {
    System.err.println("拷贝失败：" + e.getMessage());
}
```

## 最常用的文本读取方式

```java
try (BufferedReader br = new BufferedReader(new FileReader("D:\\test.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
} catch (IOException e) {
    System.err.println("读取失败：" + e.getMessage());
}
```

## 三层套娃

```
FileInputStream    — 连接文件（最底层）
InputStreamReader  — 字节→字符，指定编码（中间层）
BufferedReader     — 加缓冲 + 按行读（最外层）
```
