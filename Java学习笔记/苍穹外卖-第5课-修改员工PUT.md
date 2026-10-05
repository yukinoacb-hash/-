# 苍穹外卖 · 第 5 课：修改员工（PUT）

## 📖 知识点概述（一句话）

修改 = **用 HTTP 的 `PUT` 方法，把改动后的整条员工数据（JSON）送过去**，
SQL 用 `UPDATE ... SET ... WHERE ...`。

到这里 CRUD 就齐了：**C**reate(新增/`POST`) · **R**ead(查询/`GET`) · **U**pdate(修改/`PUT`) · **D**elete(删除/`DELETE`)。

---

## 🧠 关键要点

### 1. 四个动作的完整对照表

| 操作 | HTTP 方法 | URL | 数据放哪 | Spring 注解 | SQL |
|---|---|---|---|---|---|
| 查全部 | GET | `/employee` | 无 | `@GetMapping` | `SELECT` |
| 查一个 | GET | `/employee/1` | 路径 | `@GetMapping("/{id}")` | `SELECT ... WHERE id` |
| 新增 | POST | `/employee` | **请求体** | `@PostMapping` | `INSERT` |
| 修改 | PUT | `/employee` | **请求体（含 id）** | `@PutMapping` | `UPDATE ... WHERE id` |
| 删除 | DELETE | `/employee/2` | 路径 | `@DeleteMapping("/{id}")` | `DELETE ... WHERE id` |

### 2. 为什么新增和修改用 `@RequestBody`，查询和删除用 `@PathVariable`？

**看这次要送多少数据**：
- 送"一整条数据"→ 请求体（JSON），因为路径里塞不下
- 送"就一个 id"→ 路径，简洁清晰

### 3. 修改接口的 id 为什么要放在请求体里？

两种都能用：

```java
@PutMapping("/{id}")                        // 风格 A：id 在路径
public Result update(@PathVariable Long id, @RequestBody Employee employee)

@PutMapping                                 // 风格 B：id 在请求体（苍穹外卖采用）
public Result update(@RequestBody Employee employee)
```

苍穹外卖用**风格 B**。原因：前端编辑页面本来就拿着完整的员工对象，
id 已经是对象里的一个字段了，直接整个发过来最省事，不用拆开。

### 4. `UPDATE` 语句的固定结构

```sql
UPDATE 表名 SET 列1=值1, 列2=值2, ... WHERE 条件
      ↑ 改什么              ↑ 改谁（不能忘！）
```

`SET` 和 `WHERE` 是**成对出现**的。

---

## 📝 代码示例

```java
// ① Mapper
@Update("UPDATE employee SET name=#{name}, phone=#{phone}, sex=#{sex}, " +
        "id_number=#{idNumber}, update_time=#{updateTime} WHERE id=#{id}")
int update(Employee employee);

// ② Service 接口
int update(Employee employee);

// ② ServiceImpl —— 业务规则：修改时间由后端填
@Override
public int update(Employee employee){
    employee.setUpdateTime(LocalDateTime.now());
    return employeeMapper.update(employee);
}

// ③ Controller
@PutMapping
public Result update(@RequestBody Employee employee){
    int rows = employeeService.update(employee);
    return rows > 0 ? Result.success() : Result.error("员工不存在");
}
```

### 测试写法

```http
### 修改员工
PUT http://localhost:8080/employee
Content-Type: application/json

{
  "id": 1,
  "name": "管理员改个名",
  "phone": "13900139000",
  "sex": "男",
  "idNumber": "110101199001011234"
}
```

---

## ⚠️ 注意事项 / 常见坑

### 坑 1（最危险）：`UPDATE` 忘写 `WHERE` = 全表更新 💀

```sql
UPDATE employee SET name = '张三'              -- ✗✗✗ 全公司的人都叫张三了
UPDATE employee SET name = '张三' WHERE id=1   -- ✓
```

这是**真实世界最经典的生产事故**之一。新手练习无所谓，真到公司：
写 `UPDATE` / `DELETE` 之前，**先把 `WHERE` 写上，再写别的**。

### 坑 2：为什么 `SET` 里不更新 `password` 和 `status`
- `password`：密码有专门的重置流程（要加密、要校验旧密码），不能顺路用编辑接口改
- `status`：启用/禁用是独立操作的开关，不是"编辑资料"的一部分
- `createTime`：创建时间一旦写下就不该变（只有 `updateTime` 该变）

**判断标准：这个字段是不是"用户可以随意编辑的资料"？不是，就别放进来。**

### 坑 3：`updateTime` 由后端填，不让前端传
前端可能传 2020 年，或者根本不传。**时间这类字段永远归后端管**（和新增时的 `createTime` 同理）。

### 坑 4：新增和修改的区别不只是 SQL
| | 新增 | 修改 |
|---|---|---|
| 关键字段 | `createTime`（新建） | `updateTime`（改动） |
| id 从哪来 | 数据库自增生成 | **前端传过来**（要先知道改谁） |
| 返回判断 | `rows > 0` | `rows > 0`（0 表示"这个 id 不存在"） |

### 坑 5：PUT 也要带 `Content-Type: application/json`
忘写会得到 **415 Unsupported Media Type** —— 因为 `@RequestBody` 找不到 JSON 就没办法转成对象。

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第4课-删除员工DELETE](苍穹外卖-第4课-删除员工DELETE.md)
- 对照复习：[JDBC 实战（学生管理系统）](JavaJDBC实战.md)
