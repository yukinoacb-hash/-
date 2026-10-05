# 苍穹外卖 · 第 4 课：删除员工（DELETE）

## 📖 知识点概述（一句话）

删除 = **把"删谁"这个 id 放进 URL 路径**，用 HTTP 的 `DELETE` 方法发出去，三层各加一个方法。

**关键认知：删除和"根据 id 查询"长得几乎一样**，区别只有两点：
1. HTTP 方法从 `GET` 换成 `DELETE`
2. SQL 从 `select` 换成 `delete`

---

## 🧠 关键要点

### 1. RESTful 方法对照表（同一个路径，靠 HTTP 方法区分动作）

| 操作 | HTTP 方法 | URL | 数据放哪 | 注解 |
|---|---|---|---|---|
| 查全部 | GET | `/employee` | 无 | `@GetMapping` |
| 查一个 | GET | `/employee/1` | 路径 | `@GetMapping("/{id}")` |
| 新增 | POST | `/employee` | **请求体 JSON** | `@PostMapping` |
| 删除 | DELETE | `/employee/2` | 路径 | `@DeleteMapping("/{id}")` |
| 修改 | PUT | `/employee` | **请求体 JSON** | `@PutMapping` |

**规律**：需要"一整条数据"→ 用 POST/PUT + 请求体；只需要"一个 id"→ 用路径 + `@PathVariable`。

### 2. 加接口的固定顺序：从下往上写

```
① Mapper  —— 写 SQL（@Delete）
② Service —— 接口声明 + 实现类调用 Mapper
③ Controller —— 接请求、判断结果、返回 Result
```

**为什么是这个顺序**：底层先有"能力"，上层才能调用。反过来写，Controller 引用一个不存在的方法，IDEA 会一路报红。

### 3. `rows > 0` 的真正含义

```java
int rows = employeeService.deleteById(id);
if (rows > 0) return Result.success();
return Result.error("员工不存在");
```

`rows` 是**影响行数**（和 insert 完全一样）。

**删一个不存在的 id，MySQL 不报错，只返回 0 行** —— 所以"删除失败"本质上就是"这个 id 找不到"，
两种情况用的是同一个判断。

---

## 📝 代码示例

```java
// ① Mapper
@Delete("DELETE FROM employee WHERE id=#{id}")
int deleteById(Long id);

// ② Service 接口
int deleteById(Long id);

// ② ServiceImpl
@Override
public int deleteById(Long id){
    return employeeMapper.deleteById(id);
}

// ③ Controller
@DeleteMapping("/{id}")
public Result<Void> deleteById(@PathVariable Long id){
    int rows = employeeService.deleteById(id);
    if (rows > 0) {
        return Result.success();
    }
    return Result.error("员工不存在");
}
```

### 测试文件写法（IDEA HTTP Client）

```http
### 新增员工
POST http://localhost:8080/employee
Content-Type: application/json

{
  "username": "zhangsan",
  "name": "张三",
  "password": "123456"
}

### 删除员工
DELETE http://localhost:8080/employee/2
```

**格式规则**：
- `###` 是**请求分隔符**，多个请求之间必须写
- 第一行：`方法 URL`
- 接着是请求头（如 `Content-Type: application/json`）
- **空一行**，再写请求体（DELETE / GET 没有请求体，两行就结束）

---

## ⚠️ 注意事项 / 常见坑

### 坑 1（本课踩到）：`DELETE * FROM employee` ← 语法错误！

```java
@Delete("DELETE * FROM employee WHERE id=#{id}")   // ✗ 错的
@Delete("DELETE FROM employee WHERE id=#{id}")     // ✓ 对的
```

**为什么错**：`*` 的意思是"所有**列**"，而**列是查询才有的概念**。
- `SELECT *` → 我要看所有列 ✅
- `DELETE` → 删除的最小单位是**整行**，没有"删某几列"这种操作 ❌

MySQL 会直接报 `You have an error in your SQL syntax ... near '* FROM employee'`。

**一句话记住**：`*` 只属于 `SELECT`。

### 坑 2：浏览器地址栏只能发 GET
DELETE 无法在浏览器里测，必须用 `.http` 文件或 Postman。

### 坑 3：`.http` 文件别放在 `src/main/java` 里
放在 `src/main/java/com/xxx/` 下面，它不是 Java 源码，会打乱包结构。
**正确位置：项目根目录** `D:\project\sky-takeout\employee.http`。

### 坑 4：返回值类型
删除**没有数据要返回**，写 `Result<Void>` 比 `Result<Employee>` 更准确
（`Result.success()` 是泛型方法，写成 `Result<Employee>` 也能编译，但语义上骗了阅读代码的人）。

### 坑 5：删除是真删，别手滑
`DELETE FROM employee WHERE id=1` 会把 admin 删掉。测试时先 `GET /employee` 看一眼 id，
**确认要删哪个再删**。（生产项目里常用"逻辑删除"代替物理删除，后面做菜品时会讲到。）

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第3课-新增员工POST](苍穹外卖-第3课-新增员工POST.md)
- 对照复习：[JDBC 实战（学生管理系统）](JavaJDBC实战.md)
