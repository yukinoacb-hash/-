# 苍穹外卖 — 第 2 课：根据 id 查询员工（GET /employee/{id}）

## 📖 知识点概述

上一课做的是"查全部"：`GET /employee` → 返回 List。
这一课做"查一个"：`GET /employee/1` → 只返回 id 为 1 的那个员工。

改动位置一共 4 处，**从下往上写**（先写最底层的 Mapper，每写一层，上面那层才有东西可以调）：

```
Mapper（写 SQL）
   ↑ 被调用
Service 接口（声明方法）
   ↑ 被调用
ServiceImpl（把请求转给 Mapper）
   ↑ 被调用
Controller（接收 URL 上的 id）
```

## 🧠 关键要点

### 1. `#{}` 和 `${}` 的区别（最重要）

```java
@Select("SELECT * FROM employee WHERE id = #{id}")   // ✅ 预编译占位符
@Select("SELECT * FROM employee WHERE id = ${id}")   // ❌ 字符串拼接，有 SQL 注入风险
```

`#{}` 底层就是 JDBC 里的 `PreparedStatement` 的 `?`，MyBatis 帮你把值安全地塞进去。
`${}` 是直接把值拼进 SQL 字符串 —— 等于回到 `Statement` 时代，能被注入。

> 串起来：JDBC 学的 `PreparedStatement` → MyBatis 里的 `#{}`，是同一个东西。

### 2. `@PathVariable` vs `@RequestParam`

| 写法 | URL | 注解 |
|------|-----|------|
| 路径变量 | `/employee/1` | `@PathVariable` |
| 查询参数 | `/employee?id=1` | `@RequestParam` |

`@PathVariable` 取的是 **路径里的一段**，所以 `@GetMapping("/{id}")` 和 `@PathVariable Long id` 名字要对应上。
（`/employee` 这个"查全部"的接口不能写成 `@GetMapping` 无参 + `@GetMapping("/{id}")` 冲突，两者路径不同所以没事。）

### 3. 返回值从 List 变成单个对象

```java
Result<List<Employee>>   // 查全部
Result<Employee>         // 查一个
```

### 4. 查不到怎么办

`SELECT` 查不到记录时 MyBatis 返回 `null`，不会报错。
所以要判断一下，返回 `Result.error("员工不存在")`，而不是把 null 塞进 success 里。

## 📝 代码示例（4 处改动）

### ① Mapper

```java
@Select("SELECT * FROM employee WHERE id = #{id}")
Employee findById(Long id);
```

### ② Service 接口

```java
Employee findById(Long id);
```

### ③ ServiceImpl

```java
@Override
public Employee findById(Long id) {
    return employeeMapper.findById(id);
}
```

### ④ Controller

```java
@GetMapping("/{id}")
public Result<Employee> findById(@PathVariable Long id) {
    Employee employee = employeeService.findById(id);
    if (employee == null) {
        return Result.error("员工不存在");
    }
    return Result.success(employee);
}
```

测试：浏览器打开 `http://localhost:8080/employee/1`

## ⚠️ 注意事项 / 常见坑

- `@GetMapping("/{id}")` 括号里的路径不能漏，光写 `@GetMapping` 会变成"查全部"，导致启动报映射冲突
- `@PathVariable` 的值名字必须和 `{id}` 里的一致，否则要写成 `@PathVariable("id") Long employeeId`
- Service 接口加了方法，实现类必须加 `@Override` 实现，否则接口和实现类不一致会编译报错
- `#{}` 括号里写的是 **实体类的属性名**（`id`），不是数据库列名
- 记得 `import org.springframework.web.bind.annotation.PathVariable;`

## 🔗 相关链接

- 第 1 课：[苍穹外卖-第1课-三层架构.md](苍穹外卖-第1课-三层架构.md)
- JDBC 里的 PreparedStatement：[JavaJDBC.md](JavaJDBC.md)
