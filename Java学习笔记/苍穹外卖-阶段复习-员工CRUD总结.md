# 苍穹外卖 · 阶段复习：员工模块 CRUD 总结

> 复习日期：2026-10-05 ｜ 覆盖第 1~5 课

---

## 一、知识地图（这一阶段到底学了什么）

```
一个 HTTP 请求的一生
─────────────────────────────────────────────
浏览器/IDEA .http                                    
     │  ① DELETE http://localhost:8080/employee/2
     ▼
┌─ Spring Boot 内嵌 Tomcat（8080 端口）─────────────┐
│  ② 按 URL + 方法 找到对应的处理方法                 │
│     路径 /employee/2  +  DELETE                    │
│       ↓                                            │
│  ③ EmployeeController.deleteById(@PathVariable id) │  ← 表现层
│       ↓  调用                                       │
│  ④ EmployeeService.deleteById(id)（接口）           │  ← 业务层
│       ↓  真正的实现                                  │
│  ⑤ EmployeeServiceImpl.deleteById(id)              │
│       ↓  调用                                       │
│  ⑥ EmployeeMapper.deleteById(id)（接口）            │  ← 数据层
│       ↓  注解 / XML 里的 SQL                        │
│  ⑦ DELETE FROM employee WHERE id = 2               │
│       ↓  HikariCP 连接池                            │
│  ⑧ MySQL（sky_takeout 库）                          │
└────────────────────────────────────────────────────┘
     │  ⑨ Result{code:1, msg:"操作成功", data:null}
     ▼
   前端拿到 JSON
```

---

## 二、自测 12 问（先自己答，再看下面答案）

1. `@RestController` 和 `@Controller` 的区别？用错会怎样？
2. `@Autowired` 到底做了什么？为什么不用 `new`？
3. `@Mapper` 和 `@MapperScan` 分别是什么？少了 `@MapperScan` 报什么错？
4. `#{}` 和 `${}` 的区别？哪个安全？
5. CRUD 四个动作分别对应哪个 HTTP 方法、哪个注解、数据放哪？
6. `@PathVariable` / `@RequestParam` / `@RequestBody` 分别从哪拿数据？
7. 为什么新增时 `createTime` 由后端填，不让前端传？
8. 为什么新增后返回 `Result.success()` 而不是 `Result.success(employee)`？
9. `UPDATE` 忘写 `WHERE` 会发生什么？
10. 状态码 404 / 405 / 415 / 500 / Connection refused 分别代表什么？
11. 为什么改完 Java 代码必须重新编译 + 重启？
12. 为什么 `username` 重复插入会 500？（提示：表结构里有个东西）

---

## 三、答案与解析

**1. `@RestController` vs `@Controller`**
`@RestController` = `@Controller` + `@ResponseBody`，方法返回值直接变成 JSON。
用 `@Controller` 的话，返回值会被当成**视图名**去渲染页面 → 报 `Circular view path` 500。

**2. `@Autowired`**
告诉 Spring："把容器里这个类型的对象给我"。
不用 `new` 是因为对象由 Spring 统一创建管理（IoC），而且 Service 里注入 Mapper、Controller 里注入 Service 的链条是 Spring 帮你接的（DI）。

**3. `@Mapper` 和 `@MapperScan`**
- `@Mapper`：标在 Mapper 接口上，告诉 MyBatis "这个接口是数据层"
- `@MapperScan("com.sky.skytakeout.mapper")`：标在启动类上，**扫描整个包**
- 少了扫描 → 启动报 `required a bean of type 'EmployeeMapper' that could not be found`

**4. `#{}` 和 `${}`**
- `#{}` → PreparedStatement 的 `?` 占位符，**安全**，99% 用它
- `${}` → 字符串直接拼接，**有 SQL 注入风险**，只有表名/列名这类"不能当参数"的地方才用

**5. CRUD 对照表**

| 操作 | HTTP | 注解 | 数据放哪 | SQL |
|---|---|---|---|---|
| 查全部 | GET | `@GetMapping` | 无 | `SELECT` |
| 查一个 | GET | `@GetMapping("/{id}")` | 路径 | `SELECT ... WHERE id` |
| 新增 | POST | `@PostMapping` | 请求体 JSON | `INSERT` |
| 修改 | PUT | `@PutMapping` | 请求体（含 id） | `UPDATE ... WHERE id` |
| 删除 | DELETE | `@DeleteMapping("/{id}")` | 路径 | `DELETE ... WHERE id` |

**6. 三种取参数方式**
`@PathVariable` 从 URL 路径（`/employee/1`）｜`@RequestParam` 从问号后（`?name=张`）｜`@RequestBody` 从请求体 JSON。

**7. 时间字段为什么后端填**
前端可能传错、可能不传（就是 null）。`createTime`/`updateTime` 属于**系统字段**，永远由后端在 Service 里 `LocalDateTime.now()` 写。
（注意：数据库的 `ON UPDATE CURRENT_TIMESTAMP` 只在"SQL 没显式赋值"时才生效。）

**8. 新增返回值**
`insert` 后对象里的 `id` 是数据库自增生成的，**此时 Java 对象里还是 null**；
而且对象里带着 `password`，返回给前端等于**泄露密码**。
判断标准：返回的数据是否**完整、安全、有明确用途**。

**9. `UPDATE` 忘写 `WHERE`**
全表更新（全表数据被改成同一个值）—— 真实世界最经典的生产事故。
**`UPDATE` / `DELETE` 先写 `WHERE`，再写别的。**

**10. 状态码**

| 状态码 | 含义 | 常见原因 |
|---|---|---|
| Connection refused | 服务没启动 | 端口没人监听（连 TCP 都失败） |
| 404 | 路径不存在 | URL 错 / 没被扫描 |
| 405 | 路径在，方法不对 | 用错 HTTP 方法 / 方法没注册（比如写成 private 或没编译） |
| 415 | 媒体类型不支持 | POST/PUT 忘了 `Content-Type: application/json` |
| 500 | 服务器内部错误 | SQL 报错、唯一索引冲突、空指针 |

**11. 为什么要重新编译**
IDEA 跑的是 `target/classes` 里的 **class 文件**，不是 `.java` 源码。
改了源码不重新构建，Spring 还是按旧 class 启动 → 新接口"不存在"（表现为 405）。
**固定节奏：改代码 → ⬛ 停止 → ▶ 重新运行。**（XML 也一样要重新构建）

**12. 为什么 username 重复会 500**
建表语句里有 `UNIQUE KEY username (username)` —— **唯一索引**。
重复插入被数据库拒绝 → `Duplicate entry 'zhangsan' for key 'employee.username'` → 抛异常 → 500。
**排查姿势：去 IDEA 控制台看堆栈，重点看最后的 `Caused by:`。**

---

## 四、项目现状（2026-10-05 快照）

### 文件
| 文件 | 状态 |
|---|---|
| `SkyTakeoutApplication.java` | `@SpringBootApplication` + `@MapperScan` |
| `common/Result.java` | 泛型 `Result<T>`：`success()` / `success(data)` / `error(msg)` |
| `entity/Employee.java` | `@Data`，10 个字段 |
| `mapper/EmployeeMapper.java` | 5 个方法（findAll / findById / insert / deleteById / update），**全部注解版** |
| `service/EmployeeService.java` | 5 个方法声明 |
| `service/EmployeeServiceImpl.java` | 新增填 status/createTime/updateTime；修改填 updateTime |
| `controller/EmployeeController.java` | 5 个接口，全部 `public` |
| `application.yml` | datasource + `mapper-locations: classpath:mapper/*.xml` + 驼峰映射 |

### 数据库 `sky_takeout.employee`
```
id=1  admin     管理员     status=1
id=7  zhangsan  张三       status=1
```
（`username` 有唯一索引；`id` 自增，失败也会跳号）

---

## 五、遗留未完成

1. **`EmployeeMapper.update` 还是"全字段覆盖式"**：
   `SET username=#{username}, name=#{name}, ...`
   → 前端只传部分字段时，`username` 会变成 null → `NOT NULL` 报 500
   → **正解就是动态 SQL（第 6 课）**
2. **动态 SQL（XML）还没动手**：`src/main/resources/mapper/EmployeeMapper.xml` 还没建
3. **修改接口还没实测过**（数据库里 `update_time` 还停在插入时间）
4. 备注：`employee.http` 还在 `src/main/java/com/sky/skytakeout/` 下，建议挪到项目根目录

---

## 六、下一步路线

```
✅ 员工 CRUD（还差动态 SQL + 实测修改）
   ↓
⬜ 登录功能 + JWT（密码 MD5 加密、拦截器）
   ↓
⬜ 分类 / 菜品 / 套餐 业务
   ↓
⬜ Redis（缓存、验证码）
   ↓
⬜ JUnit 单元测试专题
   ↓
⬜ RAG 智能客服（PGVector / Elasticsearch + OpenAI API）
   ↓
⬜ Java Agent
```

---

## 🔗 相关链接

- [第 1 课：三层架构](苍穹外卖-第1课-三层架构.md)
- [第 2 课：根据 id 查询](苍穹外卖-第2课-根据id查询.md)
- [第 3 课：新增员工 POST](苍穹外卖-第3课-新增员工POST.md)
- [第 4 课：删除员工 DELETE](苍穹外卖-第4课-删除员工DELETE.md)
- [第 5 课：修改员工 PUT](苍穹外卖-第5课-修改员工PUT.md)
- [第 6 课：动态 SQL](苍穹外卖-第6课-动态SQL(MyBatis%20XML).md)
