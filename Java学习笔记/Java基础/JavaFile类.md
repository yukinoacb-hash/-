# File 类

## 📖 知识点概述
File 是文件或文件夹路径的抽象表示。注意：new File() 只是创建了一个"路径对象"，硬盘上不一定有这个文件！

## 🧠 关键要点

### 三种构造方式
```java
File f1 = new File("D:\\test.txt");              // 绝对路径
File f2 = new File("D:\\学习笔记", "test.txt");  // 父路径 + 子路径
File f3 = new File(new File("D:\\学习笔记"), "test.txt"); // File对象 + 子路径
```

### 常用方法
**判断类：**
```java
f.exists();        // 是否存在
f.isFile();        // 是否是文件
f.isDirectory();   // 是否是文件夹
```

**获取类：**
```java
f.getName();          // 文件名
f.length();           // 文件大小（字节）
f.getAbsolutePath();  // 绝对路径
```

**创建删除类：**
```java
f.createNewFile();  // 创建文件
f.mkdir();          // 创建单级文件夹
f.mkdirs();         // 创建多级文件夹（最常用）
f.delete();         // 删除（不经过回收站！）
```

**遍历类：**
```java
String[] names = dir.list();     // 文件名数组
File[] files = dir.listFiles();  // 文件对象数组
```

### 递归遍历骨架
```java
public static void list(File dir) {
    File[] files = dir.listFiles();
    if (files == null) return;        // 判空
    for (File f : files) {
        System.out.println(f.getName());
        if (f.isDirectory()) {
            list(f);  // 递归
        }
    }
}
```

## ⚠️ 注意事项 / 常见坑
1. listFiles() 可能返回 null，一定要判空
2. delete() 直接删除，不经过回收站
3. "." 代表当前目录，".." 代表上级目录
