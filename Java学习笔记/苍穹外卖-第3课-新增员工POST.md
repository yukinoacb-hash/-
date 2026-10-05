# 苍穹外卖 · 第 3 课：新增员工（POST）+ 405 排查

## 📖 知识点概述（一句话）

**GET 是"我来拿数据"，POST 是"我来送数据"。** 送数据时 JSON 写在请求体（body）里，
Spring 用 `@RequestBody` 把它变成 Java 对象，再交给 Service → Mapper 写进数据库。

三层分工（和学生管理系统完全一样）：

| 层 | 这层只负责 | 本课代码 |
|---|---|---|
| Controller | 收请求、做最简单的判断、返回结果 | `@PostMapping` + `@RequestBody` |
| Service | 业务规则（状态、时间、校验、加密） | 补 `status=1`、`createTime`、`updateTime` |
| Mapper | 一条 SQL | `@Insert` 插入 9 列（不写 id，数据库自增） |

---

## 🧠 关键要点

### 1. `@PostMapping` 对应 HTTP 的 POST
- `@GetMapping` → 查，`@PostMapping` → 增，`@PutMapping` → 改，`@DeleteMapping` → 删
- 这四兄弟统称 RESTful 风格：**同一个路径 `/employee`，靠 HTTP 方法区分动作**

### 2. `@RequestBody`（只用在 POST/PUT）
把请求体里的 JSON **反序列化**成 Java 对象，类似 JDBC 里把 `ResultSet` 变成 `Employee` 的过程，
只不过这次是 Spring + Jackson 帮你做。

```java
@PostMapping
public Result insert(@RequestBody Employee employee){
    int rows = employeeService.insert(employee);   // rows = 影响行数（和 JDBC 一模一样）
    return rows > 0 ? Result.success() : Result.error("添加失败");
}
```

### 3. 为什么 Service 里要补 `status` / `createTime` / `updateTime`
前端只发了"员工是谁"，**"这条记录什么时候创建、状态是什么"属于后端业务规则**，
不该让前端传，也不该让前端决定。所以放在 Service，统一由后端填。

### 4. 返回值用 `Result.success()` 而不是 `Result.success(employee)`
- `insert` 时 `id` 是数据库自增生成的，此刻对象里 `id` 还是 `null`（除非加 `@Options(useGeneratedKeys=true, keyProperty="id")` 回填）
- 对象里还带着 `password`，返回给前端等于**泄露密码**
- 前端新增后通常只需要知道"成功了没"，然后重新拉一次列表

**判断标准：返回的数据是否完整、安全、有明确用途。三者缺一就别返回。**

---

## ⚠️ 注意事项 / 常见坑

### 坑 1（本课踩到）：405 Method Not Allowed = 路径找到了，但方法不对
```
{"status":405,"error":"Method Not Allowed","path":"/employee"}
```
含义：`/employee` 这个路径**存在**（因为有 `@GetMapping`），但**没有注册 POST 的处理方法**。

**本课真实原因：代码改了，但没有重新编译/重启。**
- 源码 `EmployeeController.java` 修改时间 14:17
- 编译产物 `target/classes/.../EmployeeController.class` 时间 14:04（更早）
- 用 `javap` 一看，旧 class 里只有 `findAll` / `findById`，**根本没有 `insert`**

👉 **改完 Java 代码必须：停止应用 → 重新编译（Rebuild）→ 再启动**。IDEA 跑的是 `target/classes` 里的 class，不是你的 .java 源码。

### 坑 2：Controller 的方法要写 `public`
```java
@PostMapping
private Result insert(...)   // ✗ 不规范，别赌框架行为
@PostMapping
public  Result insert(...)   // ✓ 统一写 public
```

### 坑 3：Connection refused（连接被拒绝）= 服务压根没启动

```
io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: localhost/[0:0:0:0:0:0:0:1]:8080
```

**含义**：8080 端口上没有程序在监听，TCP 连都连不上。**和 404/405 完全是两回事**。

| 报错 | 谁的问题 | 说明 |
|---|---|---|
| Connection refused | 服务没起来 | 端口没人监听 |
| 404 / 405 | 服务活着 | 到了 Spring，路径/方法没匹配上 |
| 415 | 服务活着 | POST 没写 `Content-Type: application/json` |
| 500 | 服务活着 | 到了你的代码，抛异常了 |

**排查顺序：先能连上 → 再看 404/405 → 再看 500。**

启动成功的标志（控制台输出）：
```
Started SkyTakeoutApplication in 3.xxx seconds
```

- 想确认端口有没有人监听：`netstat -ano | findstr :8080`
- `localhost` 可能被解析成 IPv6 的 `::1`，若仍 refused，改用 `127.0.0.1:8080`

**开发循环**：改代码 → ⬛ 停止 → ▶ 重新运行 → 测试。少一步就会测到旧代码或连不上。
### 一个项目里同时注入同一个 Service 两次
`Controller` 注入 `EmployeeService`、`EmployeeServiceImpl` 又注入 `EmployeeService`（自己）→ 循环依赖。
正确写法是 `EmployeeServiceImpl` 注入 `EmployeeMapper`。

---

## 🔍 排错三件套（状态码语义）

| 状态码 | 含义 | 常见原因 |
|---|---|---|
| 404 | 路径不存在 | URL 写错、Controller 没被扫描到 |
| 405 | 路径在，方法不对 | 用 GET 打了 POST 接口，或处理方法没注册 |
| 415 | 媒体类型不支持 | POST 没写 `Content-Type: application/json` |
| 500 | 服务器内部错误 | SQL 报错、空指针、循环依赖 |
| 400 | 参数/JSON 格式不对 | JSON 字段类型和实体对不上 |

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第2课-根据id查询](苍穹外卖-第2课-根据id查询.md)
- 对照复习：[JDBC 实战（学生管理系统）](JavaJDBC实战.md)
