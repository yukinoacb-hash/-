# IO流

## 📖 知识点概述
I = Input（输入），O = Output（输出）。IO流用于从文件读数据和往文件写数据。

## 🧠 关键要点

### IO流分类
```
IO流
 ├── 字节流（读写所有文件：图片、视频、Word...）
 │    ├── FileInputStream    （读）
 │    └── FileOutputStream   （写）
 └── 字符流（读写纯文本文件：txt、java...）
      ├── FileReader         （读）
      └── FileWriter         （写）
```

### 字节流读文件
```java
// 一次读一个字节（慢）
FileInputStream fis = new FileInputStream("D:\\test.txt");
int b;
while ((b = fis.read()) != -1) {
    System.out.print((char) b);
}

// 一次读一个字节数组（快）
FileInputStream fis = new FileInputStream("D:\\test.txt");
byte[] buffer = new byte[1024];
int len;
while ((len = fis.read(buffer)) != -1) {
    System.out.print(new String(buffer, 0, len));
}
```

### 字节流写文件
```java
// 覆盖写
FileOutputStream fos = new FileOutputStream("D:\\test.txt");
fos.write("Hello".getBytes());

// 追加写（加true参数）
FileOutputStream fos = new FileOutputStream("D:\\test.txt", true);
fos.write("追加内容".getBytes());
```

### 文件拷贝（核心练习）
```java
try (FileInputStream fis = new FileInputStream("D:\\src.jpg");
     FileOutputStream fos = new FileOutputStream("D:\\dest.jpg")) {
    byte[] buffer = new byte[1024];
    int len;
    while ((len = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, len);
    }
}
```

### try-with-resources（自动关流）
```java
// JDK 7+ 推荐写法，自动调用 close()，不用写 finally
try (FileInputStream fis = new FileInputStream("D:\\test.txt")) {
    int b;
    while ((b = fis.read()) != -1) {
        System.out.print((char) b);
    }
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

## ⚠️ 注意事项 / 常见坑
1. 流用完一定要 close()，否则文件会被占用
2. FileOutputStream 默认是覆盖写，追加要加 true 参数
3. 字节流读中文会乱码，读文本用字符流（FileReader）
