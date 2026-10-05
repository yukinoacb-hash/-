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

## ⚠️ 注意事项 / 常见坑

- **JWT 不是加密**：payload 谁都能看，只保证"改不了"
- **token 里别放敏感信息**：密码、身份证、手机号都不行
- **一定要设过期时间**：不然这张票永远有效，泄露了收不回来
- **注销很麻烦**：JWT 一旦发出，到期前服务器管不住它（真实项目要配 Redis 做"黑名单"，到时候讲）

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第8课-登录与JWT](苍穹外卖-第8课-登录与DTO.md)