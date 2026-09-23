# 异常处理

## 遇到异常的两条路

```java
try-catch   → 自己当场处理
throws      → 甩锅给别人处理（练习用，正式项目少用）
```

## 正式项目推荐：try-with-resources

```java
try (FileInputStream fis = new FileInputStream("D:\\test.txt")) {
    // 用 fis 读文件
} catch (IOException e) {
    System.err.println("操作失败：" + e.getMessage());
}
// try 结束后自动关流，不用手写 finally
```

## 练习和正式项目的区别

| | 练习 | 正式项目 |
|--|------|---------|
| 异常 | `throws Exception` 甩锅 | `try-catch` 自己处理 |
| 关流 | 常常忘了关 | try-with-resources 自动关 |
| 结果 | 崩了就崩了 | 出错了要友好提示 |
