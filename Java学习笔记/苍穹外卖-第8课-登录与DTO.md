# 苍穹外卖 · 第 8 课：登录（上）—— 验身份 + DTO

## 📖 知识点概述（一句话）

登录 = **验身份**（用户名密码对不对）+ **发通行证**（JWT）。
本课先做前半段，跑通 `POST /employee/login`，JWT 放到下一半。

---

## 🧠 一、"登录"这两个字里藏着两件事

| 阶段 | 干什么 | 输出 |
|---|---|---|
| ① 验身份 | 拿用户名去查库，把用户输入的密码 MD5 后和库里的比对 | 对 / 不对 |
| ② 发通行证 | 对了之后给前端一张凭证（JWT token） | 一串字符串 |

**为什么要拆成两步做？**

因为一次加两个新东西，出错了你**分不清是密码比对错了还是 token 生成错了**。

> 🎯 **程序员的基本功：一次只加一个变量。**
> 先让"验身份"这条链路跑通、能返回正确结果，再去纠结 JWT。

---

## 🧠 二、数据流（先看懂这张图）

```
前端  POST /employee/login
      body: {"username": "admin", "password": "123456"}
        ↓
Controller   接住 JSON，交给 Service（自己不干活）
        ↓
Service      ① 明文密码 → MD5
             ② 拿 username 去数据库查这个人
             ③ 两个 MD5 字符串比一比
             ④ 对 → 返回 Employee；不对 → 报错
        ↓
Mapper       SELECT * FROM employee WHERE username = #{username}
        ↓
数据库       把查到的行（password 是密文）返回
```

### 和"新增员工"对比

| | 新增员工 | 登录 |
|---|---|---|
| HTTP 方法 | POST | POST |
| 密码方向 | 明文 → **加密后存库** | 明文 → **加密后比对** |
| 返回什么 | 操作成功/失败 | 登录成功/失败（后面加工 JWT） |

**共同点：密码永远是"进后端立刻变 MD5"**，明文只在网络上和内存里活一瞬间。

---

## 🧠 三、DTO 是什么？为什么要新建 `dto` 包

### 先看你在"新增员工"里是怎么接参数的

```java
@PostMapping
public Result<Employee> insert(@RequestBody Employee employee){ ... }
```

这里**直接用 Entity 接的**——因为新增员工时前端传的字段，和数据库 `employee` 表的字段几乎一一对应。**能用，但不够好。**

### 登录为什么不能用 Employee 接？

因为它们是两种不同的东西：

| | `entity.Employee` | `dto.EmployeeLoginDTO` |
|---|---|---|
| 描述的是 | **数据库表里有什么** | **这个接口要什么参数** |
| 字段数 | 10 个（id/username/name/password/phone/sex/idNumber/status/createTime/updateTime） | **2 个**（username/password） |
| 谁能改 | 表结构变了才改 | 接口变了就改 |

**DTO = Data Transfer Object，数据传输对象。**
专门用来描述"接口的入参 / 出参长什么样"，和数据库表结构解耦。

### 用 DTO 的三个好处

1. **自我文档**：打开 `EmployeeLoginDTO` 一眼就知道"登录要传 username 和 password"，不用去翻 Employee 猜哪几个字段有用
2. **防止脏字段**：前端万一多传个 `status: 0`、`id: 999`，因为 DTO 里根本没这两个字段，Spring 直接忽略，**进不来**
3. **改动隔离**：以后登录要加"图形验证码 `code` 字段"，改 DTO 就行，**不用动 Employee、也不用动数据库表**

> 📌 约定俗成的分层：
> - `entity` —— 对应数据库表
> - `dto` —— 接口入参（前端 → 后端）
> - `vo` —— 接口出参（后端 → 前端，用来"挡掉"不想给的字段，比如 password）
>
> 你之前问"为什么 getById 会把 password 也返回给前端"——那就是缺 `vo` 的后果，后面补。

---

## 📝 代码示例（已写完 ✅）

### 第 1 步：`dto/EmployeeLoginDTO.java` —— 登录接口的入参

```java
package com.sky.skytakeout.dto;

import lombok.Data;

@Data
public class EmployeeLoginDTO {
    private String username;   // 登录用户名
    private String password;   // 明文密码（后端收到后立刻 MD5）
}
```

> 关键点：
> - `@Data`（Lombok）自动生成 getter/setter/toString
> - **不写 `@TableName`、不写数据库相关注解** —— DTO 跟数据库没关系

---

### 🔍 DTO 的写法跟哪个文件一样？

**跟 `entity/Employee.java` 一样** —— 都是"光秃秃的数据盒子"：

```java
package com.sky.skytakeout.dto;      // 只有包名跟 Employee 不同

import lombok.Data;                   // 一样

@Data                                 // 一样
public class EmployeeLoginDTO {       // 类名不同
    private String username;
    private String password;
}
```

| | `Employee` | `EmployeeLoginDTO` |
|---|---|---|
| 包 | `com.sky.skytakeout.entity` | `com.sky.skytakeout.dto` |
| 类名 | `Employee` | `EmployeeLoginDTO` |
| 字段 | 10 个 | 2 个 |
| import | `lombok.Data` + `java.time.LocalDateTime` | **只有** `lombok.Data` |

### ❌ DTO 不需要的东西（新手最容易画蛇添足）

| 不需要 | 为什么 |
|---|---|
| 接口 + 实现类（像 `EmployeeService` / `EmployeeServiceImpl`） | 那是**有行为**的组件才要；DTO 只有数据 |
| `@Service` / `@Mapper` / `@RestController` | DTO 不是"干活的 Bean" |
| `@MapperScan` 里加它 | 那是 Mapper 专用的 |
| Mapper / XML | DTO 不碰数据库 |
| 无参构造、getter/setter 手写 | `@Data` 全包了 |

### 🧩 那 DTO 是谁 new 出来的？

**Spring MVC 帮你 new 的，你不用管。**

```
前端发 JSON
   ↓
Spring 看到 @RequestBody + EmployeeLoginDTO
   ↓
Jackson（JSON 工具）反射：new EmployeeLoginDTO() → 逐个调 setUsername() / setPassword()
   ↓
把对象交给你写的 Controller 方法
```

> 这就是为什么**必须要有 setter**（`@Data` 生成）—— 没有 setter，Jackson 塞不进值，运行时直接报错。
---

## 📝 代码示例（第 2 步：Mapper 加 `getByUsername`）

### 位置：`mapper/EmployeeMapper.java`，跟 `findById` 放在一起

```java
@Select("SELECT * FROM employee WHERE username = #{username}")
Employee getByUsername(String username);
```

### 三个"为什么"

**① 为什么参数是 `String`，不是 `Employee`？**
因为登录这一刻你还**不知道这个人是谁** —— 手里只有前端传来的一个用户名。
拿用户名去换一整个人（`Employee`）出来，这才是目的。

```
String username  →（查库）→  Employee 对象（查到了）/ null（查不到）
```

**② 为什么要用 `WHERE` 查，而不是 `findAll()` 出来再 for 循环找？**

```java
// ❌ 笨办法
List<Employee> all = employeeMapper.findAll();
for (Employee e : all) { if (e.getUsername().equals(username)) ... }
```

- 100 个员工还好，10 万个员工就是**把 10 万行数据从数据库搬到内存**，纯浪费
- `username` 上有唯一索引，数据库**知道怎么秒查**；`WHERE` 就是把活儿甩给专业的干

> 🎯 一句话：**能在数据库做的筛选，就不要搬到 Java 里做。**

**③ `#{username}` 里的名字有讲究吗？**
这里只有一个参数，MyBatis 会直接把它绑进去，写成 `#{username}` 是为了**让人看懂**。
（多个参数时必须用 `@Param` 标名字，以后遇到再说。）

### 顺带解释：为什么 `SELECT *` 出来的 `id_number` 能自动变成 `idNumber`？

因为 `application.yml` 里配了这一句：

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true   # id_number → idNumber
```

数据库用**下划线**命名（`id_number`），Java 用**驼峰**命名（`idNumber`），
这个开关负责两边自动对上号 —— 你的 `findById`、`findAll` 能正常返回数据，全靠它。
---

## 📝 代码示例（第 3 步：Service 加 `login`）

### 为什么"比对密码"这件事必须写在 Service？

| 层 | 该管什么 | 不该管什么 |
|---|---|---|
| Controller | 收请求、返回结果 | ❌ 不管业务规则 |
| **Service** | **业务规则**（密码怎么比、能不能登录） | ❌ 不管 SQL 怎么写 |
| Mapper | SQL 怎么写 | ❌ 不管业务判断 |

"密码要 MD5 后再比"是一条**业务规则** —— 明天老板说"换成 BCrypt"，你只改 Service，Controller 和 Mapper 一个字不动。

### ① 接口 `EmployeeService` 加声明

```java
Employee login(EmployeeLoginDTO employeeLoginDTO);
```

（记得 `import com.sky.skytakeout.dto.EmployeeLoginDTO;`）

### ② 实现类 `EmployeeServiceImpl` 写逻辑 —— 一共 5 步

```java
@Override
public Employee login(EmployeeLoginDTO employeeLoginDTO){
    // ① 拿用户名去数据库换人
    // ② 查不到（null）→ 返回 null
    // ③ 把 DTO 里的明文密码加密（和新增时用的是同一个工具方法）
    // ④ 加密结果 跟 库里存的密文 比对，不一样 → 返回 null
    // ⑤ 一样 → 返回 employee 对象
}
```

### ③ 两个必须想明白的点

**（a）比对的方向不能反**

```java
// ✅ 对的
String md5 = DigestUtils.md5DigestAsHex(employeeLoginDTO.getPassword().getBytes());
if (!md5.equals(employee.getPassword())) { return null; }

// ❌ 错的（也行不通）
// "把数据库里的密文解密出来跟明文比" —— 解不了，也不需要
```

> 再看一遍第 7 课的结论：**摘要不可逆，所以验证密码的方式永远是"再算一遍，然后比字符串"。**
> 字符串比较用 `equals`，**不要用 `==`**（`==` 比的是"是不是同一个对象"）。

**（b）"用户不存在"和"密码错误"为什么都返回 `null`？**

因为**不能让接口告诉别人"这个用户名存在"**：

```
攻击者拿 employee / admin / zhangsan 一个个试
   ↓
如果接口回答"用户不存在" vs "密码错误"
   ↓
他就知道哪些用户名真实存在了 → 这叫"用户名枚举"，是常见攻击的第一步
```

所以业务上统一返回 null，前端统一提示 **"用户名或密码错误"**，一个字都不多给。

> ⚠️ 这里先这么写。等员工模块收尾专门讲「异常处理」时，
> 原项目的做法是抛 `AccountNotFoundException` / `PasswordErrorException`，
> 再由全局异常处理器翻译给前端 —— 效果一样，写法更专业。

### ④ 已知待补的小洞

`employeeLoginDTO.getPassword()` 如果是 `null`（前端没传密码），`.getBytes()` 会直接**空指针 500**。

> 第 7 课坑 2 已经埋过伏笔。现在先跑通主链路，回头统一加非空校验。
---

## 🕳️ 坑 6：方法声明了返回值，**所有分支都必须 return**

**现象**：`login` 方法写了一半，IDEA 上一片红，报 `缺少返回语句`。

```java
// ❌ 编译不过
@Override
public Employee login(EmployeeLoginDTO employeeLoginDTO){
    String username = employeeLoginDTO.getUsername();
    if(username == null){
        return null;
    }
    // ← 到这里方法就结束了，可是"返回值是 Employee"这句话没兑现
}
```

**规则：方法头写了 `Employee`，编译器就要求"不管走哪条路，最后都得吐出一个 Employee 出来"**
（返回 `null` 也算"吐出了一个值"，是合法的）。

所以 `if` 里面 `return null` 只解决了"这一条路"，**主流程那条路还没着落** → 编译报错。

> 🎯 写法定型：
> ```java
> public Employee login(...){
>     if (倒霉情况) { return null; }   // 提前撤退
>     ...
>     return employee;                // 最后必须有个"兜底"的返回值
> }
> ```

---

## 🕳️ 坑 7：`username == null` 的检查 —— 好心，但没打中要害

```java
if(username == null){ return null; }   // 不算错，但没用
```

**为什么"不算错"？** 提前拦住空值是好习惯（叫**防御性编程**），JDBC 时代你还得 `if (rs.next())` 一个个判断，思路是对的。

**为什么"没打中要害"？**

| 字段 | 会不会 NPE | 原因 |
|---|---|---|
| `username` | ❌ 不会 | 它只是被塞进 SQL 当参数。`WHERE username = null` 是**合法** SQL，结果就是"查不到"，最后照样返回 null |
| `password` | ✅ **会** | `.getBytes()` 是**对一个对象调用方法**，对象是 null 就直接炸 |

**区别在哪：一个是"被当作值传递"，一个是"被当作对象调用方法"。**
`null` 当值传递没问题；对 `null` **调方法**才 NPE。

> 所以真正该防的是：
> ```java
> if (employeeLoginDTO.getPassword() == null) { return null; }
> ```
> 不过这个先放着，等统一加参数校验时再补。
---

## ✅ `login` 完整代码（定稿）

```java
@Override
public Employee login(EmployeeLoginDTO employeeLoginDTO){

    // ① 拿用户名去数据库换人
    Employee employee = employeeMapper.getByUsername(employeeLoginDTO.getUsername());

    // ② 查不到 → 用户名不对，撤退
    if (employee == null) {
        return null;
    }

    // ③ 把前端传来的明文密码，用和新增时一模一样的算法加密
    String md5Password = DigestUtils.md5DigestAsHex(employeeLoginDTO.getPassword().getBytes());

    // ④ 加密结果 跟 库里存的密文 比对，不一样 → 密码不对，撤退
    if (!md5Password.equals(employee.getPassword())) {
        return null;
    }

    // ⑤ 全对上了，返回这个员工
    return employee;
}
```

### 逐行拆解

| 行 | 干了什么 | 为什么 |
|---|---|---|
| ① | `getByUsername(...)` | 用用户名换整个人；查不到返回 `null` |
| ② | `if (employee == null)` | 用户名不存在。**顺带兜住了 username 传 null 的情况** |
| ③ | `md5DigestAsHex(...getBytes())` | 和 `insert()` 里**同一套算法**——这是关键，两边不一致就永远登不上 |
| ④ | `!md5Password.equals(...)` | 注意 `!` 在**最前面**：`equals` 返回"一样吗"，我们要的是"不一样吗" |
| ⑤ | `return employee` | 给上层（Controller）一个"这个人是谁"的答案，后面发 JWT 要用 |

### 为什么第 ④ 行要写 `!`

```java
if (md5Password.equals(employee.getPassword()))  { /* 密码对，往下走 */ }
if (!md5Password.equals(employee.getPassword())) { return null; }   // 反过来，错了就撤
```

第二种叫**卫语句（guard clause）**：把"出错的情况"提前踢出去，主流程就能一路平铺到底，不用层层嵌套 `if-else`。

> 对比一下新手写法：
> ```java
> if (密码对) {
>     return employee;
> } else {
>     return null;
> }
> ```
> 也能跑，但函数一长，嵌套就会越陷越深。**能提前撤退，就别往深了套。**

### 关于 `username == null` 那一句

可以删掉 —— 第 ② 行已经合并兜住了。**需求没变，代码少了一截**，这就是"越写越干净"。
## ⚠️ 注意事项 / 常见坑

- **别拿 Entity 当 DTO 用**：能跑，但登录这种"只要两个字段"的场景会显得含糊
- **DTO 里的 password 是明文**：它只在"HTTP 请求 → Service 加密"这一段存活
- **`@Data` 别忘了**：不写就没有 getter，Jackson 反序列化 JSON 会失败（拿不到 setter）

---

## 🕳️ 坑 8（最危险的一个）：漏掉 `!`，登录逻辑整个反了

**实际写出来的代码：**

```java
// ❌ 少了 !
if(md5Password.equals(employee.getPassword())){
    return null;
}
```

**推演一下会发生什么（这是排查 bug 的基本功：把代码念出来）：**

| 用户输入 | 两个密文 | `equals` 返回 | 走到哪一步 | 结果 |
|---|---|---|---|---|
| **密码正确** | 一样 | `true` | 进 if → `return null` | ❌ **登录失败** |
| **密码错误** | 不一样 | `false` | 跳过 if → `return employee` | ✅ **登录成功！** |

> 😱 **正确密码登不上，随便输个错的反而能进** —— 这是安全漏洞里最严重的一档。

### 怎么一眼看穿它

`if (条件) { return null; }` 这种**提前撤退**的写法，条件是**"出错的情况"**。

所以读的时候要**把中文念出来**：

```
"如果 密码一样 就撤退"     ← 念出来就知道不对劲
"如果 密码不一样 就撤退"   ← 这才对
```

> 🎯 每个 `if` 都念一遍中文，是新人最有效的自检手段。

### 为什么这个 bug 特别可怕

- **编译器不报错** —— 语法完全合法
- **IDEA 不飘黄** —— 不是"可能的 NPE"，它看不出来
- **代码看着很正常** —— 缩进、括号、方法名全对

**只有真跑一次才露馅。** 这就是"为什么要自己测"的最好例子 ——
不是老师让你测，是**有些 bug 只能靠运行抓出来**。

### 正确写法

```java
if (!md5Password.equals(employee.getPassword())) {
    return null;
}
```

### 记住这三条测试用例（每次改完登录都要跑）

| 用例 | 期望结果 |
|---|---|
| `admin` / `123456`（正确的） | ✅ 登录成功 |
| `admin` / `1234567`（错密码） | ❌ 用户名或密码错误 |
| `wangwu` / `123456`（不存在的人） | ❌ 用户名或密码错误 |

> 第 2 条就是专门用来抓"`!` 漏写"这类反转 bug 的。**只测成功路径 = 没测。**
## ✅ Controller 的登录接口（定稿）

```java
@PostMapping("/login")
public Result<Employee> login(@RequestBody EmployeeLoginDTO employeeLoginDTO){
    Employee employee = employeeService.login(employeeLoginDTO);

    if(employee == null){
        return Result.error("用户名或密码错误");
    }

    return Result.success(employee);
}
```

**整条链路终于闭环了：**

```
前端  POST /employee/login  {"username":"admin","password":"123456"}
        ↓
Controller  @RequestBody → EmployeeLoginDTO（Spring 帮你 new 的）
        ↓
Service     查人 → 加密 → 比对
        ↓
Mapper      SELECT * FROM employee WHERE username = #{username}
        ↓
数据库       返回那一行
        ↓
Controller  Result.success(employee) / Result.error("用户名或密码错误")
```

### 验收：三条必须跑的测试用例

```
### ① 正确密码
POST http://localhost:8080/employee/login
Content-Type: application/json

{ "username": "admin", "password": "123456" }

### ② 密码故意输错
POST http://localhost:8080/employee/login
Content-Type: application/json

{ "username": "admin", "password": "1234567" }

### ③ 不存在的人
POST http://localhost:8080/employee/login
Content-Type: application/json

{ "username": "wangwu", "password": "123456" }
```

| 用例 | 期望结果 |
|---|---|
| ① 正确密码 | `{"code":1,"msg":"操作成功","data":{...}}` |
| ② 密码错 | `{"code":0,"msg":"用户名或密码错误"}` |
| ③ 用户不存在 | `{"code":0,"msg":"用户名或密码错误"}` |

> ⚠️ 改完代码**必须重启**再测（老规矩：跑的是 `target/classes` 里的 class）。

---

## 🧠 重要概念：HTTP 状态码 ≠ 业务状态码

登录失败时你会看到：

```
HTTP/1.1 200          ← HTTP 状态码：传输层面"请求送达了、服务器答了"
Content-Type: application/json

{
  "code": 0,          ← 业务状态码：业务层面"这次登录没成功"
  "msg": "用户名或密码错误"
}
```

| | HTTP 状态码 | 业务状态码（`Result.code`） |
|---|---|---|
| 谁定的 | HTTP 协议 | 我们自己定的 |
| 说什么 | "这次请求本身有没有走通" | "这个业务有没有办成" |
| 例子 | 200 / 404 / 405 / 500 | 1 成功 / 0 失败 |
| 谁会看 | 浏览器、运维监控 | **前端**（拿 code 决定弹什么提示） |

**为什么要分开？**

- `404` —— 路径写错了，这是**程序问题**，得程序员改
- `{"code":0,"msg":"用户名或密码错误"}` —— 路径没错、服务器没坏，只是**用户输入错了**，该让前端弹提示

> 🎯 一句话：**HTTP 状态码管"路通不通"，业务 code 管"事办没办成"。**
> 苍穹外卖全程用 HTTP 200 + 自定义 code 的方案。

---

## 🕳️ 已知漏洞（下一步就修）：登录成功把 password 返回给前端了

看用例 ① 的返回：

```json
{ "code":1, "msg":"操作成功", "data": {
    "username":"admin",
    "password":"e10adc3949ba59abbe56e057f20f883e",   ← 不该给！
    ...
}}
```

虽然是密文，但**密文也不该给前端**（拿到就能拿去撞库 / 分析）。

**正解：用 VO（View Object）** —— 后端 → 前端专用的出参对象，只放前端要的字段。

> 这就是 `vo` 包存在的意义，也是你之前问"为什么 getById 会返回 password"的答案。
## 🔗 相关链接

- 上一课：[苍穹外卖-第7课-登录与密码加密(MD5)](苍穹外卖-第7课-登录与密码加密(MD5).md)