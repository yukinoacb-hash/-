# Java 方法引用（Method Reference）

## 📖 知识点概述

**方法引用** 是 Java 8 引入的一种语法糖，它让你用更简洁的写法来表达 Lambda 表达式。

简单说：**当 Lambda 体只是直接调用一个已有的方法时，就可以用方法引用来替代，让代码更短、更清晰。**

```
Lambda:        (参数) -> 某个类或对象.方法(参数)
方法引用:      某个类或对象::方法名
```

## 🧠 关键要点

| 类型 | 语法 | 对应 Lambda |
|------|------|------------|
| **静态方法引用** | `类名::静态方法` | `(args) -> 类名.静态方法(args)` |
| **实例方法引用（特定对象）** | `对象::实例方法` | `(args) -> 对象.实例方法(args)` |
| **实例方法引用（任意对象）** | `类名::实例方法` | `(obj, args) -> obj.实例方法(args)` |
| **构造方法引用** | `类名::new` | `(args) -> new 类名(args)` |

## 📝 代码示例

### 1️⃣ 静态方法引用

```java
import java.util.Arrays;
import java.util.List;

public class StaticMethodRef {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

        // Lambda 写法
        names.forEach(name -> System.out.println(name));

        // 方法引用写法（更简洁）
        names.forEach(System.out::println);
    }
}
```

`System.out::println` 就是方法引用，`System.out` 是目标对象，`println` 是要调用的方法。

### 2️⃣ 实例方法引用（特定对象）

```java
public class InstanceMethodRef {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("apple", "banana", "cherry");

        // Lambda 写法
        names.stream()
             .map(name -> name.toUpperCase())
             .forEach(System.out::println);

        // 方法引用写法
        names.stream()
             .map(String::toUpperCase)    // 任意 String 对象的 toUpperCase
             .forEach(System.out::println);
    }
}
```

这里 `String::toUpperCase` 是 **任意对象** 的实例方法引用——意思是"把每个 String 对象调 toUpperCase()"。

### 3️⃣ 构造方法引用

```java
import java.util.*;
import java.util.stream.Collectors;

public class ConstructorRef {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

        // Lambda 写法
        List<String> list1 = names.stream()
                                  .collect(Collectors.toCollection(() -> new ArrayList<>()));

        // 方法引用写法
        List<String> list2 = names.stream()
                                  .collect(Collectors.toCollection(ArrayList::new));

        System.out.println(list2); // [Alice, Bob, Charlie]
    }
}
```

`ArrayList::new` 就是构造方法引用，等价于 `() -> new ArrayList<>()`。

### 4️⃣ 实战对比：排序

```java
import java.util.*;

public class SortExample {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("banana", "apple", "cherry", "date");

        // ❌ 匿名类（Java 8 之前）
        words.sort(new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return a.compareTo(b);
            }
        });

        // ✅ Lambda
        words.sort((a, b) -> a.compareTo(b));

        // ✅✅ 方法引用（最简洁）
        words.sort(String::compareTo);

        System.out.println(words); // [apple, banana, cherry, date]
    }
}
```

## ⚠️ 注意事项 / 常见坑

1. **方法引用不是方法调用**：`String::toUpperCase` 不会执行方法，它只是引用了方法名，等 stream 处理时才会调用
2. **参数必须匹配**：被引用的方法参数要和函数式接口的抽象方法参数一致
3. **不能滥用**：当 Lambda 体中包含多条语句或复杂逻辑时，不适合用方法引用，强行用反而降低可读性
4. **区分 `类名::静态方法` 和 `类名::实例方法`**：前者整个参数来自 Lambda 参数，后者第一个参数作为调用者，其余参数传给方法

## 🔗 相关链接 / 延伸阅读

- [Java Lambda 表达式](/Java学习笔记/Java Lambda表达式.md)（如果已整理）
