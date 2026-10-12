# 苍穹外卖 · 复习：从 MD5 加密到拦截器注册（全链路）

> 日期：2026-10-09
> 覆盖课程：第 7 课（MD5 加密）→ 第 8 课（登录验身份）→ 第 9 课（JWT）→ 第 10 课（拦截器 / ThreadLocal / 注册）

---

## 📖 一句话主线

整条链路其实只回答一个问题：

> **"怎么确认这次请求，是一个已经登录过的员工发来的？"**

为了回答它，我们依次解决了 5 个小问题：

| 顺序 | 解决什么 | 用什么 |
|---|---|---|
| ① | 密码被人看到怎么办 | MD5 摘要（存密文，不存明文） |
| ② | 怎么判断密码对不对 | 明文再算一遍 → 比对密文 |
| ③ | 服务器记不住"你登录过" | JWT（自证身份的通行证） |
| ④ | 谁来验这张通行证 | 拦截器 |
| ⑤ | 验出来的人怎么传给 Service | ThreadLocal（`BaseContext`） |
| ⑥ | 拦截器怎么才能生效 | `WebMvcConfiguration` 注册 |

---

## 🗺️ 全景图

```text
【入职】POST /employee
  Controller 收 Employee
    → Service.insert：MD5 加密 + 补 status/createTime/updateTime
      → Mapper → INSERT → 库里躺着密文
          ↓
【登录】POST /employee/login          ← 白名单放过，不需要 token
  Controller 收 DTO
    → Service.login：
         getByUsername 查人
         → 明文 MD5 再算一遍 → 和库里密文比对 → 对上
         → 造 claims{empId} → createJWT（密钥签名）→ 造 VO
    → 返回 { id, name, userName, token }
          ↓
【之后每个请求】GET /employee/7       ← 被拦截
  LoginCheckInterceptor.preHandle：
       取 header 里的 token → parseJWT 验签
       ✔ 通过：empId 存进 ThreadLocal → return true → Controller 执行
       ✘ 失败：setStatus(401) → return false → 请求到此为止
  afterCompletion：清掉 ThreadLocal
```

**灵魂细节：从头到尾，empId 都没让前端传过。** 前端只传 token，empId 是服务器自己从 token 里掏出来的 —— 前端伪造不了。

---

## 第 1 步 · MD5：密码为什么不能明文存

**类比**：明文存密码 = 把全公司的钥匙挂在门口墙上。数据库一旦被拖库，所有人的密码直接裸奔。

MD5 是**摘要算法**（哈希），三个必须记住的特性：

- **单向**：`123456` → `e10adc3949ba59abbe56e057f20f883e`，但反过来推不回去
- **定长**：不管输入多长，输出永远是 32 位十六进制
- **确定性**：同样的输入，永远得到同样的输出

👉 第三条是登录能成立的前提：**不需要"解密"，只需要"再算一遍"**。

新增员工时的加密（`EmployeeServiceImpl.insert`）：

```java
employee.setPassword(DigestUtils.md5DigestAsHex(employee.getPassword().getBytes()));
```

**为什么写在 Service 层？**

- Controller 只管收发（HTTP 层）
- Mapper 只管 SQL（数据库层）
- "怎么处理业务数据" 属于 Service

⚠️ 加密必须发生在**进数据库之前**，而且要覆盖**所有入口** —— 这就是第 7 课要清洗历史数据的原因：之前直接往库里塞的明文（比如 `lisi`），登录时算出来的密文永远比对不上。

---

## 第 2 步 · 登录：是"比对"，不是"解密"

`login` 的四个动作，本身就是一份业务判断清单：

```java
// ① 拿用户名去库里换人
Employee employee = employeeMapper.getByUsername(employeeLoginDTO.getUsername());
// ② 查不到 → 用户名不对，撤退
if (employee == null) { return null; }
// ③ 把前端传来的明文密码，用和新增时一模一样的算法加密
String md5Password = DigestUtils.md5DigestAsHex(employeeLoginDTO.getPassword().getBytes());
// ④ 加密结果 跟 库里存的密文 比对，不一样 → 密码不对，撤退
if (!md5Password.equals(employee.getPassword())) { return null; }
```

**为什么是"再算一遍"而不是"解开"？** 因为 MD5 不可逆 —— 你没法把库里的密文还原成明文。只能"你传来的明文算一遍，看结果等不等于我存的那串"。

⚠️ 这里"一模一样"是硬要求：同样的输入 + 同样的算法 → 同样的输出。换算法、加盐，全对不上。

⚠️ **安全小缺陷（黑马这套没做，但要知道）**：MD5 是**快速**哈希，通用密码可以查彩虹表。生产环境一般用 **BCrypt**（自带盐 + 故意慢）。

⭐ **设计亮点**：不管"用户名不存在"还是"密码错误"，返回的都是同一句 `"用户名或密码错误"`。这是在防 **用户枚举** —— 别让人知道哪个用户名是真实存在的。

---

## 第 3 步 · 发通行证：JWT

密码对了 ≠ 完事。**HTTP 是无状态的** —— 服务器记不住"你刚才登录过"。

两条路：服务端记账（Session）or 发一张自证的凭证（JWT）。黑马选 JWT。

签发就三行（`EmployeeServiceImpl.login` 后半段）：

```java
Map<String, Object> claims = new HashMap<>();
claims.put("empId", employee.getId());       // ① 通行证上写什么
String token = JwtUtil.createJWT(
        jwtProperties.getAdminSecretKey(),   // ② 用哪把钥匙签
        jwtProperties.getAdminTtl(),         // ③ 多久过期
        claims);                             // ④ 上面装的数据
```

`JwtUtil.createJWT` 的内部：

```java
long expMillis = System.currentTimeMillis() + ttlMillis;                        // 过期时间
SecretKey key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8)); // 字符串密钥 → HS256 钥匙
return Jwts.builder()
        .claims(claims)                          // 装 payload
        .expiration(exp)                         // 装过期时间
        .signWith(key, Jwts.SIG.HS256)           // ★签名★
        .compact();                              // 打包成字符串
```

### JWT 的三段：`header.payload.signature`

| 段 | 内容 | 能看吗 | 能改吗 |
|---|---|---|---|
| header | 声明算法（HS256） | ✅ Base64 解得出 | 改了就验不过 |
| payload | claims（empId、过期时间） | ✅ Base64 解得出 | 改了就验不过 |
| signature | 用密钥对前两段做的签名 | ❌ | — |

⚠️ **前两段是 Base64，不是加密**，贴到网上一秒就解出来。所以**绝对不能往 payload 里放密码**。JWT 防的不是"被看"，而是"被改"。

**为什么改不了？** 签名 = 用密钥 + 前两段算出来的。别人改了 payload，签名就对不上了；只有拿着密钥的服务器能重新算对。这就是 `secretKey` 的作用 —— **它是防伪图章的模具，只在服务器手里**。

### 密钥从哪来：`JwtProperties`

```java
@Component                                    // 交给 Spring 管，才能被注入
@ConfigurationProperties(prefix = "sky.jwt")  // 把 yml 里 sky.jwt.* 自动塞进来
@Data                                         // Lombok 生成 getter/setter
public class JwtProperties {
    private String adminSecretKey;
    private String adminTokenName;
    private long adminTtl;
}
```

```yaml
sky:
  jwt:
    admin-secret-key: sky-takeout-...   # 模具
    admin-ttl: 7200000                  # 2 小时（毫秒）
    admin-token-name: token             # 前端放请求头的哪个字段
```

⚠️ 起作用的是 **yml 里的值**，`JwtProperties` 只是个"搬运工"。它靠**字段名匹配**（`adminSecretKey` ↔ `admin-secret-key`，中划线自动转驼峰）。**名字写错 = 值是 null**。

### 为什么返回 VO 而不是 Employee

```java
EmployeeLoginVO vo = new EmployeeLoginVO();
vo.setId(employee.getId());
vo.setUserName(employee.getUsername());
vo.setName(employee.getName());
vo.setToken(token);
return vo;
```

- `Employee` 里躺着 `password`（密文）、`idNumber`、`sex`…… 登录页根本不需要
- **给前端的东西只给前端需要的**，多给一个字段就多一份泄露风险

三种对象的分工（务必分清）：

| 对象 | 是谁的样子 | 出现的场合 |
|---|---|---|
| **Entity** | 数据库表的样子 | 和 Mapper 打交道 |
| **DTO** | 前端交给你的样子 | Controller 收参数 |
| **VO** | 你交给前端的样子 | Controller 返数据 |

---

## 第 4 步 · 前端拿着 token 之后的每次请求

登录成功后，前端把 token 存起来（如 localStorage），之后每个请求都带上：

```http
GET /employee
token: eyJhbGciOiJIUzI1NiJ9.eyJlbXBJZCI6MSwiZXhwIjox...
```

**请求头字段叫什么名字？** 由 yml 的 `admin-token-name: token` 决定 —— 所以拦截器里写 `jwtProperties.getAdminTokenName()`，而不是硬编码 `"token"`。改配置就能改名字，不用动代码。

---

## 第 5 步 · 拦截器：谁来验这张通行证

### 为什么需要它？

⚠️ **JWT 不会自己验证自己！** token 就是个字符串，它不知道自己是真是假。必须有人拿密钥去 `parseJWT` 验一验。

而 `parseJWT` 在全项目**唯一的调用点**，就在拦截器里。这就是"既然有 JWT 了为什么还要拦截器"的答案。

### 为什么不在每个 Controller 里写一遍？

CRUD 有 5 个接口，后面还有分类、菜品、套餐…… 几十个接口。每个都写一遍验签 = 复制几十遍。拦截器 = **一处集中处理**，写一次，全部生效。

### 拦截器拦的是谁？

不是"张三这个人"。拦截器是**无状态**的 —— 它拦的是**这一次请求**。上次验过 ≠ 这次可以放行。所以每个请求都要从头验一遍。

### `preHandle` 逐行

```java
if (!(handler instanceof HandlerMethod)) {   // ① 不是 Controller 方法？放行（比如静态资源）
    return true;
}
String token = request.getHeader(jwtProperties.getAdminTokenName());  // ② 从请求头取 token

try {
    Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token); // ③ ★验签★
    Long empId = Long.valueOf(claims.get("empId").toString());                 // ④ 掏出 empId
    BaseContext.setCurrentId(empId);                                           // ⑤ 存进 ThreadLocal
    return true;                                                               // ⑥ 放行
} catch (Exception e) {
    response.setStatus(401);   // ⑦ 伪造 / 过期 / 没带 / 格式错 → 401
    return false;              //    拦下
}
```

`parseJWT` 内部：

```java
Jwts.parser().verifyWith(key).build().parseSignedClaims(token).getPayload();
```

它做三件事：① 重新算签名 ② 对比 token 里的签名 ③ 检查有没有过期。任何一项不对就**抛异常** —— 所以 catch 一网打尽。

⚠️ `catch (Exception e)` 这里**故意**不细分异常：伪造的、过期的、没带的、格式错的，处理方式都一样 —— 401。

### `afterCompletion` 那一句

```java
BaseContext.removeCurrentId();
```

请求结束了要把 ThreadLocal 清掉。**为什么？** Tomcat 的线程是**复用**的 —— 这次请求用完的线程，下次可能去伺候别人。不清就会串号：张三的请求结束，李四的请求拿到张三的 empId。

---

## 第 6 步 · ThreadLocal：empId 怎么从拦截器传到 Service

**问题**：拦截器拿到了 empId，做增删改的时候怎么用上？

- 方案一：Controller 每个方法加 `Long empId` 参数 → 签名被登录态污染，每个接口都要改 ❌
- 方案二：存全局变量 → 多线程并发下互相覆盖 ❌
- 方案三（黑马）：**ThreadLocal** ✅

**类比**：ThreadLocal 像一个**每线程专属的储物柜**。

- Tomcat 给每个请求分配一个线程（T1、T2……）
- 请求 T1 进来 → 在 T1 的柜子里放 `empId=1`
- 请求 T2 进来 → 在 T2 的柜子里放 `empId=2`
- 谁也不看谁的柜子，天然隔离

```java
public class BaseContext {
    private static final ThreadLocal<Long> threadLocal = new ThreadLocal<>();
    public static void setCurrentId(Long id) { threadLocal.set(id); }
    public static Long getCurrentId() { return threadLocal.get(); }
    public static void removeCurrentId() { threadLocal.remove(); }
}
```

| 方法 | 谁调用 | 什么时候 |
|---|---|---|
| `setCurrentId` | 拦截器 | 验签通过后 |
| `getCurrentId` | Service | 需要"当前登录人是谁"时（如自动填充 createUser/updateUser） |
| `removeCurrentId` | 拦截器 `afterCompletion` | 请求结束 |

⚠️ **不能"等 T1 结束再用 T2 的"** —— T1 和 T2 是**同时**在跑的。T2 等 T1 = 服务器一次只能伺候一个请求，那还叫什么并发。

---

## 第 7 步 · 注册：告诉 Spring "有这么个拦截器"

写完 `LoginCheckInterceptor` 并加了 `@Component`，Spring 只是**管**着它 —— 但 Spring MVC **不知道**你想让它上岗。所以需要：

```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {   // ① 声明：我来改 Spring MVC 配置
    @Autowired
    private LoginCheckInterceptor loginCheckInterceptor;          // ② 把闸机拿过来

    @Override
    public void addInterceptors(InterceptorRegistry registry) {    // ③ 重写这个"登记入口"
        registry.addInterceptor(loginCheckInterceptor)             // ④ 登记用哪个拦截器
                .addPathPatterns("/**")                            // ⑤ 拦：所有路径
                .excludePathPatterns("/employee/login");           // ⑥ 放过：登录接口
    }
}
```

### 五个必须懂的"为什么"

1. **为什么要 `@Configuration`？** 没这个注解，Spring 启动时扫不到这个类，"登记"就没人做 —— **编译全绿，但拦截器一声不响地不生效**。最阴的坑。
2. **为什么要 `implements WebMvcConfigurer`？** 这是 Spring MVC 留的配置接口。一实现它，框架启动时就**主动来调**你的 `addInterceptors` —— 不是你调的，是框架调的（和 `preHandle` 一样是回调）。
3. **为什么要放过 `/employee/login`？** 登录**本来就没 token**！不放过 → 去登录 → 没 token → 401 → **死锁：不登录拿不到 token，拿不到 token 登不了录**。
4. **链式调用怎么理解？** 三个方法都返回同一个"登记项"：
   ```java
   InterceptorRegistration reg = registry.addInterceptor(loginCheckInterceptor);
   reg.addPathPatterns("/**");
   reg.excludePathPatterns("/employee/login");
   ```
5. **`/**` 是什么意思？** 双星是为了匹配**多级路径**（`/employee/7`、`/category/page`）。

---

## 🧠 三个概念别搞混

| 概念 | 一句话 |
|---|---|
| **认证（Authentication）** | 你是谁 —— 拦截器验 token 就是在做这个 |
| **授权（Authorization）** | 你能干什么 —— **本项目还没做**（employee 表没角色字段，登录了就都能增删改） |
| **无状态** | 服务器不记你，每次请求自带凭证 |

---

## ⚠️ 排错口诀

| 现象 | 病因 |
|---|---|
| 405 | 请求方法不对 |
| 404 | 路径不存在 |
| 415 | 忘写 `Content-Type` |
| 500 | 看控制台 `Caused by:` |
| Connection refused | 服务没启动 |
| 改了没效果 | 没重启 |
| **编译绿但拦截器不生效** | 忘 `@Configuration` 或删了 `@Override` |
| **导错包** | `java.net.http.*` ❌ → `jakarta.servlet.http.*` ✅ |

---

## 📝 自测

1. 为什么登录时是"再算一遍 MD5"而不是"解开密文"？
2. 为什么返回给前端的是 VO 而不是 Employee？
3. JWT 三段里，哪一段是防篡改的关键？为什么？
4. JWT 已经有了，为什么还需要拦截器？
5. 拦截器拦的是"人"还是"请求"？
6. 为什么 `excludePathPatterns` 里必须有 `/employee/login`？
7. ThreadLocal 在这里解决了什么问题？不用它会怎样？
8. 为什么请求结束要 `removeCurrentId()`？
9. `WebMvcConfiguration` 里的方法是你主动调用的吗？
10. 认证和授权的区别？本项目缺了哪一个？

---

## 🔗 相关笔记

- [第 7 课：登录与密码加密（MD5）](苍穹外卖-第7课-登录与密码加密(MD5).md)
- [第 8 课：登录（上）—— 验身份 + DTO](苍穹外卖-第8课-登录与DTO.md)
- [第 9 课：JWT 通行证](苍穹外卖-第9课-JWT通行证.md)
- [第 10 课：拦截器 + ThreadLocal](苍穹外卖-第10课-拦截器与ThreadLocal.md)
- [复习速查：登录与 JWT](苍穹外卖-复习-登录与JWT速查.md)
---

# 🚶 分站版：从登录开始走一遍（2026-10-12 补充）

> 上一版是从 MD5 讲起，这一版倒过来：**先看结局，再一步步倒推**。

## 先看"结局"

项目跑起来，发一个**不带 token** 的请求：

```http
GET http://localhost:8080/employee
```

→ 返回 **401**。谁拦的？拦截器。

**所以你得先有一张 token。** 倒推：token 从哪来？→ 登录接口给的。

---

## 第 1 站 · POST /employee/login 进来了

```http
POST http://localhost:8080/employee/login
Content-Type: application/json

{ "username": "admin", "password": "123456" }
```

**关键点①：为什么这个请求没被拦？**
`WebMvcConfiguration` 里写了 `.excludePathPatterns("/employee/login")` —— 白名单。
登录时你手上根本没有 token，还拦 = **死锁**（不登录拿不到 token，拿不到 token 登不了录）。

**关键点②：它落在哪个方法上？** 三个信息共同决定：
- `POST` → `@PostMapping`
- `/employee/login` → 类上的 `@RequestMapping("/employee")` + 方法上的 `"/login"`
- `Content-Type: application/json` → 才能被 `@RequestBody` 接住

```java
@PostMapping("/login")
public Result<EmployeeLoginVO> login(@RequestBody EmployeeLoginDTO employeeLoginDTO){
```

**为什么用 DTO 而不是 Employee？** 登录只需要"用户名 + 密码"两样，DTO 精确表达"这次前端只交这两样"。

---

## 第 2 站 · Controller → Service

```java
EmployeeLoginVO employeeLoginVO = employeeService.login(employeeLoginDTO);
```

**Controller 为什么不自己查库？** 三层分工 —— Controller 只管收发，业务判断（密码对不对、要不要发证）全在 Service。

`login` 的四个动作 = 一份业务判断清单：

```java
// ① 拿用户名去库里换人
Employee employee = employeeMapper.getByUsername(employeeLoginDTO.getUsername());
// ② 查不到 → 用户名不对，撤退
if (employee == null) { return null; }
// ③ 明文密码用同样的算法再算一遍
String md5Password = DigestUtils.md5DigestAsHex(employeeLoginDTO.getPassword().getBytes());
// ④ 和库里的密文比对，不一样 → 密码不对，撤退
if (!md5Password.equals(employee.getPassword())) { return null; }
```

⭐ 两种失败返回**同一句** `"用户名或密码错误"` → 防用户枚举。

---

## 第 3 站 · 密码对了，开始发证

```java
Map<String, Object> claims = new HashMap<>();
claims.put("empId", employee.getId());       // 证上写什么
String token = JwtUtil.createJWT(
        jwtProperties.getAdminSecretKey(),   // 用哪把钥匙签
        jwtProperties.getAdminTtl(),         // 多久过期
        claims);
```

**签名的意义**：前两段（header/payload）是 Base64，**能看不能改** —— 改了签名就对不上。

```java
EmployeeLoginVO vo = new EmployeeLoginVO();   // 拿一张"对外名片"
vo.setId(employee.getId());
vo.setUserName(employee.getUsername());
vo.setName(employee.getName());
vo.setToken(token);                           // 通行证也塞进名片
return vo;
```

---

## 第 4 站 · 前端拿到 token

```json
{
  "code": 1,
  "msg": "操作成功",
  "data": { "id": 1, "userName": "admin", "name": "管理员", "token": "eyJ..." }
}
```

前端把 `data.token` 存起来（localStorage 之类），之后每个请求带上：

```http
GET /employee
token: eyJhbGciOiJIUzI1NiJ9...
```

⚠️ 字段名叫 `token`，是因为 yml 里 `admin-token-name: token`。**改配置就能改名字，代码不用动。**

---

## 第 5 站 · 请求回来，撞上闸机

### 为什么必须有闸机？

⚠️ **JWT 不会自己验证自己。** token 就是个字符串。必须有人拿密钥去 `parseJWT` 验 —— 而 `parseJWT` 全项目**唯一**的调用点，就在拦截器里。

### 拦截器的三个时刻

| 方法 | 时机 | 我们用它干嘛 |
|---|---|---|
| `preHandle` | 进 Controller **之前** | 验票，决定放行还是拦下 |
| `postHandle` | Controller 之后、渲染之前 | 没用 |
| `afterCompletion` | 请求**彻底结束** | 打扫卫生（清 ThreadLocal） |

### preHandle 逐行

```java
if (!(handler instanceof HandlerMethod)) {   // ① 不是 Controller 方法？放行（如静态资源）
    return true;
}
String token = request.getHeader(jwtProperties.getAdminTokenName());  // ② 取 token

try {
    Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token); // ③ ★验签★
    Long empId = Long.valueOf(claims.get("empId").toString());                 // ④ 掏出 empId
    BaseContext.setCurrentId(empId);                                           // ⑤ 存进 ThreadLocal
    return true;                                                               // ⑥ 放行
} catch (Exception e) {
    response.setStatus(401);   // ⑦ 伪造/过期/没带/格式错
    return false;              //    拦下
}
```

`parseJWT` 内部：① 重算签名 ② 对比 token 里的签名 ③ 检查过期 —— 任一不对就**抛异常**，所以 catch 一网打尽。

---

## 第 6 站 · empId 存哪儿 —— ThreadLocal

拦截器拿到了 empId，Service 怎么用上？→ 存进**每线程专属的储物柜**。

- T1 请求进来 → 在 T1 的柜子里放 `empId=1`
- T2 请求进来 → 在 T2 的柜子里放 `empId=2`
- 互不干扰

```java
setCurrentId  ← 拦截器验签通过后放进去
getCurrentId  ← Service 要用"当前登录人"时取出来
removeCurrentId ← afterCompletion 里清掉
```

⚠️ 请求结束必须清。Tomcat 线程**复用**，不清就会串号。

---

## 第 7 站 · 闸机怎么上岗

```java
@Configuration                       // 不加它 → 编译全绿但拦截器一声不响不生效
public class WebMvcConfiguration implements WebMvcConfigurer {  // 实现它 → 框架主动来调你
    @Autowired
    private LoginCheckInterceptor loginCheckInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginCheckInterceptor)   // 登记
                .addPathPatterns("/**")                  // 拦所有
                .excludePathPatterns("/employee/login"); // 放过登录
    }
}
```

---

## 📋 全程一张表

| 站点 | 发生了什么 | 谁在做 |
|---|---|---|
| 1 | 收 JSON → DTO | Controller |
| 2 | 查人 + 比对密码 | Service |
| 3 | 签发 token + 造 VO | Service |
| 4 | 存 token，之后每个请求带上 | 前端 |
| 5 | 验签，通过放行 / 失败 401 | 拦截器 preHandle |
| 6 | empId 存进 ThreadLocal | 拦截器 / BaseContext |
| 7 | 声明"拦哪些、放过哪些" | WebMvcConfiguration |
| 8 | 请求结束清 ThreadLocal | 拦截器 afterCompletion |

## 📝 分站自测

1. 为什么登录接口必须进白名单？
2. `@RequestBody` 能接住 JSON，前提是什么？
3. 为什么两种失败都返回同一句 "用户名或密码错误"？
4. token 为什么不能伪造？哪一段在防伪？
5. 拦截器的三个时刻分别是什么？我们用了哪两个？
6. 为什么 `handler instanceof HandlerMethod` 要放行其它类型？
7. 请求结束为什么必须 `removeCurrentId()`？
8. 少了 `@Configuration` 会怎样？
---

# 🧩 为什么要拦截器 —— 一个看得见的例子

## 没有拦截器的世界

`EmployeeController` 有 5 个接口，那每个都得自己验票：

```java
@GetMapping
public Result<List<Employee>> findAll(HttpServletRequest request) {
    // ↓↓↓ 验票代码，每个接口抄一遍 ↓↓↓
    String token = request.getHeader("token");
    Claims claims;
    try {
        claims = JwtUtil.parseJWT("sky-takeout-...", token);
    } catch (Exception e) {
        return Result.error("未登录");
    }
    Long empId = Long.valueOf(claims.get("empId").toString());
    // ↑↑↑ 这 9 行 × 5 个接口 ↑↑↑

    return Result.success(employeeService.findAll());
}
```

## 三个真实后果

**① 迟早会漏一个（最危险）**

```java
@DeleteMapping("/{id}")
public Result<Void> deleteById(@PathVariable Long id) {
    employeeService.deleteById(id);        // ← 忘了抄验票代码
    return Result.success();
}
```

编译通过、接口能跑、测试"正常"，但**这个接口裸奔**：谁都能删员工。漏一个 = 一个洞。

**② 改一次要改 30 处**

密钥换了 / 请求头名换了：
- 有拦截器 → 改 **1 处**
- 没拦截器 → 改 **30+ 处**（后面还有分类、菜品、套餐、订单），漏 1 处就废

**③ 接口签名被污染**

每个方法都得多一个 `HttpServletRequest` 参数 —— 登录态不该出现在业务方法的参数表里。

## 有拦截器的世界

那 9 行验票代码**一个字都不用写在 Controller 里**：

```java
@GetMapping
public Result<List<Employee>> findAll() {     // 干干净净，只管查
    return Result.success(employeeService.findAll());
}

@DeleteMapping("/{id}")
public Result<Void> deleteById(@PathVariable Long id) {   // 新接口自动被保护
    int rows = employeeService.deleteById(id);
    return rows > 0 ? Result.success() : Result.error("员工不存在");
}
```

验票只有**一处** —— `LoginCheckInterceptor.preHandle`。

## 生活类比：写字楼

楼里 30 个房间，每个房间 = 一个接口。

- **没拦截器**：每个房间自己派人查工牌 → 30 个人重复干同一件事，总有人偷懒/漏掉；换工牌样式要通知 30 个人
- **有拦截器**：一楼大门口**一个闸机**，刷卡才进 → 标准统一，谁也漏不了；新增房间不用管门禁

**白名单**同理：登录接口必须在闸机**外面**（一楼大厅的访客登记处）。不然你还没卡就进不去登记处，永远拿不到卡。

**ThreadLocal** 同理：闸机刷完卡，把"你是谁"塞进你口袋（每线程一个口袋）；里面房间要用就掏口袋。

## 还有哪些事属于拦截器

"每个接口都要做、但和具体业务无关"的事：

- 验登录
- 记录请求日志 / 耗时
- 限流（同一 IP 一秒 100 次就拒）
- 打印请求参数方便排查

写在 Controller 里 = 这四件事 × 30 个接口 = 120 份重复代码。

## ⭐ 一句话

拦截器不是为了"少写几行"，是为了**保证"必做之事：一处写、处处生效、谁都漏不了"**。