# 方法引用（Method Reference）

## 有啥用？

**让 Lambda 更短。** 当 Lambda 里只调了一个现成方法时，直接用 `::` 代替，少写一堆啰嗦的 `(参数) ->`。

## 举个例子

```java
// 匿名类 —— 又臭又长
list.sort(new Comparator<String>() {
    public int compare(String a, String b) { return a.compareTo(b); }
});

// Lambda —— 好一点了
list.sort((a, b) -> a.compareTo(b));

// 方法引用 —— 一句话搞定
list.sort(String::compareTo);
```

## 一句话记法

> `对象::方法名` = "用这个对象去调这个方法"

比如 `System.out::println` 就是"用 System.out 去调用 println"
