# Java 异常处理（Exception）

## 📖 知识点概述
异常是程序运行中出现的"不正常情况"。Java 把异常设计成对象，通过 try-catch 机制让程序更健壮。

## 🧠 关键要点

### 异常体系结构
```
Throwable（最顶层）
 ├── Error（严重错误，如内存溢出，程序无法处理）
 └── Exception（异常，程序可以处理）
      ├── RuntimeException（运行时异常，编译时不检查）
      │    ├── NullPointerException
      │    ├── ArrayIndexOutOfBoundsException
      │    ├── ArithmeticException
      │    └── IllegalArgumentException（最常用！）
      └── 其他异常（编译时异常，必须处理）
           ├── IOException
           └── FileNotFoundException
```

### 编译时异常 vs 运行时异常
| 类型 | 检查时机 | 要不要处理 |
|------|---------|-----------|
| 运行时异常 | 运行时报错 | 可以不处理，但建议修复代码 |
| 编译时异常 | 写代码就报错 | 必须处理，否则编译不通过 |

### try-catch（接住异常）
```java
try {
    int[] arr = {1, 2, 3};
    System.out.println(arr[5]);
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("数组越界了！");
    e.printStackTrace();
}
// 程序继续往下走 ✅
```

### throw（主动扔异常）
```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("年龄不能为负");
    }
}
```

### throw vs throws
| | throw | throws |
|--|-------|--------|
| 位置 | 方法体内部 | 方法声明末尾 |
| 后面跟 | 异常对象 | 异常类型 |
| 含义 | 主动扔一个异常 | 声明可能会抛出异常 |

### 自定义异常
```java
public class BalanceNotEnoughException extends RuntimeException {
    public BalanceNotEnoughException() {}
    public BalanceNotEnoughException(String message) {
        super(message);
    }
}
```

### 异常转换
```java
// 编译时异常 → 运行时异常（更方便传播）
try {
    FileInputStream fis = new FileInputStream("D:\\test.txt");
} catch (FileNotFoundException e) {
    throw new RuntimeException(e);
}
```

## ⚠️ 注意事项 / 常见坑
1. try-catch 不会自动修复错误，catch 里要自己写修复逻辑
2. catch 异常时尽量写具体类型，不要全用 RuntimeException
3. 业务逻辑层多用 throw，控制层多用 try-catch
