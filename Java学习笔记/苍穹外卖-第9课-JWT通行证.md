# 苍穹外卖 · 第 9 课：JWT 通行证

## 📖 知识点概述（一句话）

登录成功后，服务器要发一张**盖了章、改不了**的通行证（JWT）给前端；
之后前端每次请求都带着它，服务器一验章就知道"你是谁"——**全程不用在服务器上存任何东西**。

---

## 🧠 一、先看"没它就不行"的问题

你已经能登录成功了。但登录成功之后呢？

```
POST /employee/login   {"username":"admin","password":"123456"}
                       → {"code":1,"msg":"操作成功","data":{...}}

GET  /employee         → ???   服务器凭什么让你看？
```

**问题：HTTP 是"无状态"的。**

> 每次请求都是**一次全新的见面**。服务器不会自动记得"你上次是谁"。
> 这不是缺陷，是设计 —— 无状态才能扛得住海量并发请求（每台服务器都能处理任意一个请求）。

**那服务器怎么才能记住"你已经登录过"？** 两条路：

| 方案 | 打个比方 | 缺点 |
|---|---|---|
| **Session** | 服务器给你一个**储物柜号码牌**，东西存在服务器柜子里 | ① 服务器要占内存，人一多就爆<br>② 多台服务器各存一份，对不上（要搞"会话共享"）<br>③ 手机 App / 跨域场景很别扭 |
| **JWT** | 给你一张**盖了章的通行证，你自己揣兜里** | 服务器不用存任何东西，但**"注销"麻烦**（票还没到期，收不回来） |

> 苍穹外卖用的是 JWT。真实的互联网产品两种都在用，看场景。

---

## 🧠 二、JWT 长什么样

**三段，用点号 `.` 分隔：**

```
eyJhbGciOiJIUzI1NiJ9 . eyJlbXBJZCI6MSwiZXhwIjoxNzMyfQ . dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
      header                    payload                          signature
```

| 段 | 装什么 | 例子（解码后） |
|---|---|---|
| **header** | 我用什么算法签的 | `{"alg":"HS256","typ":"JWT"}` |
| **payload** | 真正的内容（"你是谁"） | `{"empId":1,"exp":1732000000}` |
| **signature** | 签名（防篡改） | 一串乱码 |

### ⚠️ 最重要的一点：**前两段是明文！**

Base64 是**编码**，不是**加密**。任何拿到 token 的人，都能解开前两段看到里面写了什么。

> 随便找个网站（jwt.io）把 token 粘进去，payload 就出来了。

**所以：JWT 里绝对不能放密码、身份证号这类东西。**
它保证的是"**改不了**"，而不是"**看不见**"。

---

## 🧠 三、为什么伪造不了？

**签名是这么算出来的：**

```
signature = HMAC-SHA256( header + "." + payload , 密钥 )
```

**密钥只有服务器知道**（写在服务器配置文件里，不发出去）。

### 攻击者能做和不能做的事

| 想干什么 | 结果 |
|---|---|
| 把 payload 里的 `empId` 从 1 改成 999 | ❌ 改完签名对不上了（签名是根据**原来的**内容算的） |
| 自己重新算一个新签名 | ❌ 没有密钥，算不出来 |
| 自己伪造一个 header + payload | ❌ 同上，签不出合法签名 |
| 看 payload 里写了啥 | ✅ 能看（所以别放敏感信息） |
| 拿别人的 token 直接用 | ✅ 能（这叫"重放"，所以 token 要设过期时间） |

### 服务器怎么验证？（这就是"不用存东西"的秘密）

```
收到 token
   ↓
用自己手里的密钥，把前两段重新算一遍签名
   ↓
跟 token 里带的签名比
   ↓
一致 ✅ + 没过期 ✅  →  认你
```

**不用查数据库、不用查内存** —— 密钥一算就知道真假。

---

## 📝 四、动手第 1 步：加 jjwt 依赖

**`pom.xml` 的 `<dependencies>` 里加这三段：**

```xml
<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### 为什么是三个包？

| 包 | 干什么 | 什么时候需要 |
|---|---|---|
| `jjwt-api` | **你写代码时要用的接口**（`Jwts`、`JwtBuilder`） | 编译期 |
| `jjwt-impl` | 真正的实现（干活的） | **运行期** |
| `jjwt-jackson` | 用 Jackson 做 JSON 转换 | **运行期** |

> 这就是"面向接口编程"的实物版：**你写代码只认接口（api），谁干活（impl）运行时再挂上。**
> 所以后两个是 `<scope>runtime</scope>` —— 编译时用不着它们，跑起来才需要。

### ⚠️ 为什么不照网上老教程用 `jjwt 0.9.1`？

| | 0.9.1（老教程） | 0.12.5（我们用） |
|---|---|---|
| 依赖数量 | 1 个 | 3 个 |
| 支持 JDK | Java 8 时代 | **JDK 21 ✅** |
| 问题 | 内部用了 `javax.xml.bind`，**Java 11 之后从 JDK 里删掉了**，一跑就报 `ClassNotFoundException` | 正常 |

> 🎯 教训：**照抄网上教程前先看它是什么年代的。** JDK 版本一升级，很多老代码直接跑不动。

### 加完之后

IDEA 右上角会弹出 Maven 提示，点 **"加载 Maven 更改"**（或者右键 `pom.xml` → Maven → 重新加载项目）。

> 这一步不能省 —— 不加依赖，后面写 `import io.jsonwebtoken.Jwts;` 会直接飘红。

---

---

## 📝 五、动手第 2 步：把 JWT 参数配到 yml + 建 `JwtProperties`

### 1. 为什么配置不能写死在 Java 里？

```java
// ❌ 写死
String secretKey = "sky-takeout-secret";
long ttl = 7200000;
```

三个理由：

| 理由 | 说明 |
|---|---|
| **密钥不该进 Git** | 代码会提交到仓库；密钥一旦进仓库，等于全网公开（还很难彻底删掉） |
| **不同环境值不同** | 开发环境 token 可以 2 小时，生产可能 30 分钟；密钥当然也不一样 |
| **改配置不用重新编译** | 改 yml → 重启就行，不用动一行 Java 代码 |

> 🎯 **规矩：凡是"会变的值"和"秘密"，都放配置文件。**

### 2. `application.yml` 加这一段

```yaml
sky:
  jwt:
    admin-secret-key: sky-takeout-admin-secret-key-please-change-me-2026
    admin-ttl: 7200000          # token 有效期，单位毫秒（7200000 = 2 小时）
    admin-token-name: token     # 前端把 token 放在请求头的哪个字段里
```

> `sky` 是**自己起的前缀**（项目名），用来跟 Spring 自带的配置（`spring.`、`server.`）区分开，
> 一看就知道这是"我们项目自己的配置"。

### 3. ⚠️ 密钥长度有硬要求：**至少 32 个字符**

jjwt 0.12.5 对 HS256 有检查，密钥太短会直接抛 `WeakKeyException`。

**为什么？** HS256 的密钥太短（比如 `itcast` 才 6 个字符），攻击者可以**把可能的密钥全试一遍**，
一旦试出来就能自己签发任意 token —— **签名的防护就彻底失效了**。

> 顺便记住：老教程里写 `admin-secret-key: itcast` —— 在 jjwt 0.12.5 + JDK 21 上**跑不起来**，
> 就是这个原因。

### 4. 建 `properties/JwtProperties` 类

**位置**：`com.sky.skytakeout.properties.JwtProperties`（新建 `properties` 包）

```java
package com.sky.skytakeout.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component                                  // 交给 Spring 管，才能被 @Autowired 注入
@ConfigurationProperties(prefix = "sky.jwt") // 把 yml 里 sky.jwt.* 的值自动塞进下面字段
@Data                                        // getter / setter
public class JwtProperties {
    private String adminSecretKey;   // ← sky.jwt.admin-secret-key
    private long adminTtl;           // ← sky.jwt.admin-ttl
    private String adminTokenName;   // ← sky.jwt.admin-token-name
}
```

### 5. 核心机制：yml 的"横杠"怎么自动对上 Java 的"驼峰"

```yaml
sky:
  jwt:
    admin-secret-key: xxx     # 横杠风格 kebab-case
```

```java
private String adminSecretKey; // 驼峰风格 camelCase
```

**它俩怎么对上的？** Spring Boot 的**宽松绑定（relaxed binding）**：

```
admin-secret-key
      ↓ 去掉横杠，后面字母大写
adminSecretKey     ✅ 自动匹配
```

> 所以字段名**不用**写成 `admin_secret_key` 或 `admin-secret-key`，
> 照 Java 习惯写驼峰就行。这是 Spring Boot 一大便利。
> ⚠️ 但也意味着**拼错字段名不会报错**，只是值是 `null` —— 排查时先怀疑这里。

### 6. 和 `@Value` 比，为什么要用 `@ConfigurationProperties`

```java
// 写法 A：@Value，值分散在各处
@Value("${sky.jwt.admin-secret-key}") private String secretKey;
@Value("${sky.jwt.admin-ttl}")        private long ttl;
@Value("${sky.jwt.admin-token-name}") private String tokenName;

// 写法 B：@ConfigurationProperties，一个类全包
@Component
@ConfigurationProperties(prefix = "sky.jwt")
@Data
public class JwtProperties { ... }
```

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| 用完就走 | ✅ 简单场景够用 | 稍重 |
| 一堆配置项 | ❌ 满屏都是 `@Value` | ✅ 集中在一个类里 |
| 复用 | ❌ 每个类都要重写一遍 | ✅ 注入这一个对象就够了 |
| 类型转换 | 手动 | 自动（`long`、`boolean`、`List` 都能转） |

> **结论：一两个值用 `@Value`；一组相关配置（JWT 这种）用 `@ConfigurationProperties`。**

### 7. 怎么确认配对了？

**没有输出 = 还没法确认。** 下一步写 `JwtUtil` 时会 `@Autowired JwtProperties`，
到时候如果字段是 `null`，多半就是这里写错了（前缀不对 / 字段名拼错）。
---

## 📝 六、动手第 3 步：`JwtUtil` —— 真正生成 token 的地方

**位置**：`com.sky.skytakeout.util.JwtUtil`（新建 `util` 包）

### 1. 这个类要干两件事

| 方法 | 什么时候用 | 干什么 |
|---|---|---|
| `createJWT` | **登录成功后** | 把"你是谁"打包 + 盖章 → 一串字符串 |
| `parseJWT` | **以后每次请求**（第 10 课拦截器） | 拆开字符串 + 验章 → 看这是谁 |

### 2. 为什么写成"静态工具类"，不写成 `@Component`

跟 `DigestUtils` 一个套路：

```java
DigestUtils.md5DigestAsHex("123456".getBytes());   // 类名.方法名()，不用 new，不用注入
JwtUtil.createJWT(secretKey, ttl, claims);          // 一样
```

**因为它是"无状态"的** —— 给什么参数就算什么，自己不记任何东西。
这种"纯计算"的工具最适合静态方法。

> 对比：`EmployeeServiceImpl` 有状态（依赖 `employeeMapper`），所以必须交给 Spring 管。
> **判断标准：这个类需要"记住什么"或者"依赖别的 Bean"吗？需要 → `@Component`；不需要 → 静态工具类。**

### 3. 生成 token 的 4 步

```
① 准备内容 claims（一个 Map：要往通行证上写什么）
② 算过期时间 = 现在 + 有效期
③ 把 yml 里的字符串密钥 → 变成 HS256 能用的 SecretKey
④ 组装 + 盖章 + 压成字符串
```

### 4. 完整代码

```java
package com.sky.skytakeout.util;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.Map;

public class JwtUtil {

    /**
     * 生成 JWT
     * @param secretKey 密钥（从 yml 来）
     * @param ttlMillis 有效期（毫秒）
     * @param claims    要装进 token 的自定义数据，比如 empId
     */
    public static String createJWT(String secretKey, long ttlMillis, Map<String, Object> claims) {

        // ① 过期时间 = 当前时间 + 有效期
        long expMillis = System.currentTimeMillis() + ttlMillis;
        Date exp = new Date(expMillis);

        // ② 把字符串密钥变成 HS256 能用的 SecretKey
        SecretKey key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));

        // ③ 组装 + 签名 + 打包
        return Jwts.builder()
                .claims(claims)                   // 装内容
                .expiration(exp)                  // 设过期时间
                .signWith(key, Jwts.SIG.HS256)    // 用密钥盖章
                .compact();                       // 压成一串字符串
    }

    /**
     * 解析 JWT（第 10 课拦截器用）
     * @param secretKey 密钥
     * @param token     前端传来的 token
     * @return 里面装的内容
     */
    public static Claims parseJWT(String secretKey, String token) {
        SecretKey key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));

        return Jwts.parser()
                .verifyWith(key)                  // 用密钥验章
                .build()
                .parseSignedClaims(token)         // 解析
                .getPayload();                    // 取出内容
    }
}
```

### 5. 逐行解释

| 代码 | 说明 |
|---|---|
| `System.currentTimeMillis()` | 当前时间戳（毫秒，`long`）—— 所以 ttl 也要用 `long` |
| `new Date(expMillis)` | jjwt 的 `expiration()` 收的是 `java.util.Date`（老日期类）。**不用纠结，JWT 库就这么设计的** |
| `secretKey.getBytes(StandardCharsets.UTF_8)` | 字符串 → 字节数组。**规范写法要指定编码**（第 7 课提过） |
| `Keys.hmacShaKeyFor(...)` | 把字节数组变成 HS256 能吃的 `SecretKey`。**密钥不足 32 字节这里直接抛异常** |
| `.claims(claims)` | 往通行证上写内容（就是那个 Map） |
| `.expiration(exp)` | 印上有效期 |
| `.signWith(key, Jwts.SIG.HS256)` | **盖章**。`Jwts.SIG.HS256` 是 0.12.x 的写法（老版本叫 `SignatureAlgorithm.HS256`） |
| `.compact()` | 把三部分拼成 `xxx.yyy.zzz` 的字符串 |
| 解析链 `parser() → verifyWith() → build() → parseSignedClaims() → getPayload()` | 0.12.x 的新写法（老版本是 `parser().setSigningKey().parseClaimsJws().getBody()`） |

> **为什么 0.12.x 把 `parseClaimsJws` 改名叫 `parseSignedClaims`？**
> 因为 JWT 分两种：**JWS**（签名的，我们要的）和 **JWE**（加密的）。
> 新名字把"我在解析一个签名过的 token"说清楚了。

### 6. 新旧 API 对照表（看教程时对不上就查这里）

| 用途 | 老版本 0.9.1 | **我们用的 0.12.5** |
|---|---|---|
| 装内容 | `setClaims(map)` | `claims(map)` |
| 设过期 | `setExpiration(date)` | `expiration(date)` |
| 签名 | `signWith(SignatureAlgorithm.HS256, key)` | `signWith(key, Jwts.SIG.HS256)` |
| 生成 | `compact()` | `compact()`（没变） |
| 解析 | `parser().setSigningKey(key).parseClaimsJws(t).getBody()` | `parser().verifyWith(key).build().parseSignedClaims(t).getPayload()` |

> ⚠️ 这就是为什么**照抄老教程会一片飘红** —— 方法名都改了。

### 7. 为什么用 `Jwts.SIG.HS256` 而不是 `.signWith(key)`

`.signWith(key)` 会根据**密钥长度自动挑**算法（32 字节 → HS256、48 字节 → HS384、64 字节 → HS512）。
自动挑虽然方便，但**"到底用了哪个算法"就变成隐藏行为了** —— 明文写出来更清楚。
## 📝 七、动手第 4 步：登录成功 → 发 token（第 9 课收尾）

前 3 步都在"造工具"（依赖 / 配置 / `JwtUtil`），这一步才真正**把它插进登录流程**。

### 1. 先补一个类：`vo/EmployeeLoginVO`

**问题**：原来 Service 直接 `return employee;` —— 整个 `Employee`（**带密码密文、手机号、身份证**）跟着 `data` 一起发给前端了。

**DTO 和 VO 是一对（方向相反）：**

| | 方向 | 干什么 | 例子 |
|---|---|---|---|
| **DTO** | 前端 ➜ 后端 | **收**参数 | `EmployeeLoginDTO`（收 username / password） |
| **VO** | 后端 ➜ 前端 | **发**数据 | `EmployeeLoginVO`（发 id / userName / name / token） |

> 记忆：**D**TO = **D**eliver in（送进来）；**V**O = **V**iew（给前端看的）。

```java
package com.sky.skytakeout.vo;

import lombok.Data;

@Data
public class EmployeeLoginVO {
    private Long id;
    private String userName;   // 故意大写 N —— 前端字段约定
    private String name;
    private String token;
}
```

> 只有 4 个字段、**没有 password** —— 靠"少给字段"来防信息泄露。

### 2. Service：密码对了以后"发牌 + 换包装"

- `EmployeeService` 接口：`login` 返回类型 `Employee` ➜ `EmployeeLoginVO`
- `EmployeeServiceImpl`：多注入一个 `JwtProperties`

```java
    @Autowired
    private JwtProperties jwtProperties;

    @Override
    public EmployeeLoginVO login(EmployeeLoginDTO employeeLoginDTO) {
        Employee employee = employeeMapper.getByUsername(employeeLoginDTO.getUsername());
        if (employee == null) return null;

        String md5Password = DigestUtils.md5DigestAsHex(employeeLoginDTO.getPassword().getBytes());
        if (!md5Password.equals(employee.getPassword())) return null;

        // 密码对了 → 发牌
        Map<String, Object> claims = new HashMap<>();
        claims.put("empId", employee.getId());     // 牌上只写"你是谁"

        String token = JwtUtil.createJWT(
                jwtProperties.getAdminSecretKey(),
                jwtProperties.getAdminTtl(),
                claims);

        // 换包装：把该给前端看的字段装进 VO
        EmployeeLoginVO vo = new EmployeeLoginVO();
        vo.setId(employee.getId());
        vo.setUserName(employee.getUsername());
        vo.setName(employee.getName());
        vo.setToken(token);
        return vo;
    }
```

| 代码段 | 干嘛 | 比喻 |
|---|---|---|
| `claims` + `createJWT` | 生成通行证 | **办证** |
| `new EmployeeLoginVO()` + 一堆 `set` | 挑字段、装牌 | **换包装** |

> `Employee` = 内部档案（带密码）；`EmployeeLoginVO` = 对外名片（只印该给人看的）。

### 3. Controller：跟着换类型

```java
    @PostMapping("/login")
    public Result<EmployeeLoginVO> login(@RequestBody EmployeeLoginDTO employeeLoginDTO){
        EmployeeLoginVO employeeLoginVO = employeeService.login(employeeLoginDTO);
        if (employeeLoginVO == null) {
            return Result.error("用户名或密码错误");
        }
        return Result.success(employeeLoginVO);
    }
```

逻辑没变，只是数据形状从 `Employee` 换成 `EmployeeLoginVO`。Controller 只管"接住 → 包 `Result` → 发出去"。

### 4. 验证（重启后 `POST /employee/login`，admin / 123456）

```json
{
  "code": 1,
  "msg": "操作成功",
  "data": {
    "id": 1,
    "userName": "admin",
    "name": "管理员",
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJlbXBJZCI6MSwiZXhwIjox..."
  }
}
```

**检查点**：`data` 里 **没有** `password` / `phone` / `idNumber` ✅，且有 `token` ✅。

### 5. 本步新踩的坑

| # | 坑 | 现象 | 正解 |
|---|---|---|---|
| 16 | Javadoc 开头少写一个星（写成 `/*`） | `@param` 报"无法解析符号" | 文档注释必须 `/**` 开头 |
| 17 | 改了接口 `login` 的返回类型，Controller 没同步改 | Controller 编译标红 | 接口 / 实现 / 调用方三处类型一起改 |

### 🧩 第 9 课一句话总结

> 登录成功后用 `createJWT` 发一张"**盖了章、改不了、会过期**"的通行证；用 `EmployeeLoginVO` 只把该给的字段发出去。
> **验票（`parseJWT`）留给第 10 课的拦截器。**

## ⚠️ 注意事项 / 常见坑

- **JWT 不是加密**：payload 谁都能看，只保证"改不了"
- **token 里别放敏感信息**：密码、身份证、手机号都不行
- **一定要设过期时间**：不然这张票永远有效，泄露了收不回来
- **注销很麻烦**：JWT 一旦发出，到期前服务器管不住它（真实项目要配 Redis 做"黑名单"，到时候讲）

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第8课-登录与JWT](苍穹外卖-第8课-登录与DTO.md)