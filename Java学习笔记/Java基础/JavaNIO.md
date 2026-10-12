# NIO（新 IO）

## 旧 IO → NIO 对照

```
File f = new File()        →  Path p = Paths.get()
手动循环拷贝               →  Files.copy() 一行搞定
自己写 while 循环读文本     →  Files.readAllLines()
递归遍历目录               →  Files.walk()
```

## 常用操作

```java
// 读所有行
List<String> lines = Files.readAllLines(Paths.get("D:\\test.txt"), StandardCharsets.UTF_8);
lines.forEach(System.out::println);

// 写文件
Files.write(Paths.get("D:\\test.txt"), "Hello NIO".getBytes());

// 拷贝文件
Files.copy(Paths.get("D:\\src.jpg"), Paths.get("D:\\dest.jpg"), StandardCopyOption.REPLACE_EXISTING);

// 遍历目录
Files.walk(Paths.get("D:\\文件夹")).forEach(System.out::println);
```

## 注

`Files.readAllLines()` 内部已经自动关流，不需要 try-with-resources。
