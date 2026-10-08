# 苍穹外卖 第 10 课：拦截器 + ThreadLocal

> **前置**：第 9 课已经能"发 JWT 通行证"了 —— 登录成功返回一个 token。
> **本课要解决的问题**：登录之后的其它接口，凭什么知道"现在操作的是谁"？

---

## 📖 知识点概述

第 9 课只做到了"发证"。但一张证发出去，**必须有人查证**，否则等于没有。

本课干三件事：

1. **拦截器（Interceptor）** —— 在请求到达 Controller 之前，统一检查 token（门口的保安）
2. **BaseContext + ThreadLocal** —— 验完 token 得到的 empId 存在哪（给每个线程发一个储物柜）
3. **WebMvcConfiguration** —— 把拦截器注册进 Spring MVC，并放行登录接口

---

## 🧠 关键要点

### 一、拦截器是什么

拦截器是 Spring MVC 提供的一个"关卡"，它在**请求到达 Controller 之前**执行你的代码。

一次请求的完整流程：

```
浏览器/Postman
   ↓
DispatcherServlet（总调度）
   ↓
Interceptor.preHandle()   ← 我们的关卡在这里，返回 true 才继续
   ↓
Controller → Service → Mapper → 数据库
   ↓
Interceptor.afterCompletion()   ← 请求结束后（清理现场）
```

> 类比：公司大门的保安。刷卡（token）→ 保安核对 → 放行或拦下。
> 跟登录接口不冲突：登录接口是你"去办卡"的地方，办卡当然不用先刷卡，所以要**放行**。

### 二、拦截器要做的 5 件事

```
① 从请求头里取 token        request.getHeader("token")
② 没 token                  → 拦下，返回未登录
③ 有 token，验签            JwtUtil.parseJWT(密钥, token)
④ 验失败（假牌/过期）        → 拦下，返回未登录
⑤ 验成功                    → 从 claims 抠出 empId，存进 BaseContext
   请求结束时                 → BaseContext.removeCurrentId()
```

### 三、为什么要用多线程

不是你 `new Thread()`，是 **Tomcat 的线程池**在做。

- Tomcat 启动时准备一个线程池（默认约 200 个线程）
- 每来一个 HTTP 请求，就从池里拿一个空闲线程处理，处理完还回去
- 所以 Controller 方法**每次可能跑在不同线程上**

为什么必须这样？因为后端请求大量时间在**等待**：

| 请求在干什么 | 要等多久 |
|---|---|
| 查数据库 | 几毫秒 ~ 几秒 |
| 调用 OpenAI API | 1 ~ 30 秒（做 RAG 时会遇到） |
| 上传图片 | 看文件多大 |

单线程的话：第一个人调 OpenAI 要等 10 秒，**这 10 秒全站所有人卡死**。
多线程让"A 在等的时候，B 能继续被处理"。

> CPU 一般一次只执行一个线程，靠快速切换制造"同时"的错觉。
> 但线程**等待时不吃 CPU**，所以"大量等待"的场景多线程收益极大。

### 四、为什么 Controller 里不带 empId

empId 的来源：登录时塞进 JWT 的 payload → 每次请求带 token → 拦截器验完抠出来。

那为什么不写成方法参数？

```java
@PostMapping
public Result insert(@RequestBody Dish dish, Long empId) {   // ❌ 不这样写
}
```

**原因 1：拦截器和 Controller 之间没有"传数据的通道"**

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
```

它只能返回 `true`（放行）/ `false`（拦下）—— 只回答"让不让过"。
而且 Spring MVC 是先执行 `preHandle` 判断，放行**之后**才调用 Controller、给参数赋值，
拦截器这时想塞也塞不进去了。

**原因 2：真正需要 empId 的不是 Controller，是 Service / Mapper**

empId 典型用途是**公共字段自动填充**：新增菜品要记 `create_user = 当前登录员工id`。
填值发生在 Service 层，最终写进 SQL 的是 Mapper 层。

如果不用 ThreadLocal，就得一路传参：

```java
dishService.save(dish, empId);    // 每个方法都多一个参数
dishMapper.insert(dish, empId);   // 方法签名被污染
```

用 ThreadLocal，任意一层随手可取，**参数 0 个**：

```java
Long empId = BaseContext.getCurrentId();
```

**原因 3：不是每个接口都需要 empId**

`GET /employee` 查列表根本不需要知道"当前是谁"。
强行加参数就是给一堆不需要的方法塞噪音。

**原因 4：这正是 ThreadLocal 的设计目的**

ThreadLocal 天生就是为"在同一个请求的整条调用链里随手传递上下文数据"而生的。
empId 就是典型的"请求上下文"。

### 五、多线程 + ThreadLocal 是一对

如果用一个普通静态变量：

```java
public static Long currentId;   // ❌ 灾难
```

```
线程 T1 处理 admin 的请求    → currentId = 1
线程 T2 处理 zhangsan 的请求 → currentId = 7   ← 把 1 覆盖了
线程 T1 接着往下跑，读 currentId → 读到 7 ❌
结果：admin 的操作被记成是 zhangsan 干的
```

ThreadLocal = **给每个线程发一个专属格子**：

```
线程 T1 的格子：1     ← 只有 T1 看得到
线程 T2 的格子：7     ← 只有 T2 看得到
```

T1 无论何时 `get()` 都拿到 1，绝不会被 T2 影响。

> 类比：static 变量像挂在墙上的**公用白板**，谁都能改；
> ThreadLocal 像**更衣室的储物柜**，每个线程一把钥匙，只开自己的柜子。

### 六、线程复用 → 必须 remove（重要坑）

Tomcat 的线程是**复用**的：T1 处理完 admin 的请求，马上可能去处理下一个请求。

如果没清柜子，T1 格子里还留着 `empId = 1`，下一个请求（哪怕没登录）就可能读到 1。

所以 `afterCompletion` 里必须：

```java
BaseContext.removeCurrentId();   // 下班清柜子
```

| 机制 | 解决什么问题 |
|---|---|
| ThreadLocal 每线程一份 | 解决**同时**发生的请求互相串号 |
| `remove()` | 解决**先后**复用线程留下残留数据 |

一个管横向隔离，一个管纵向清理，两个都少不了。

### 七、Controller 到底用不用 BaseContext

- **大多数情况不用** —— empId 主要给 Service 自动填充公共字段
- **少数情况会用** —— 比如"查询我的订单"，Controller 里也可以 `BaseContext.getCurrentId()`
- 但注意：用的是 `BaseContext.getCurrentId()`，**不是方法参数 `Long empId`**

---

## 📝 代码示例

### BaseContext（ThreadLocal 的包装类）

```java
package com.sky.skytakeout.common;

public class BaseContext {

    private static final ThreadLocal<Long> threadLocal = new ThreadLocal<>();

    public static void setCurrentId(Long id) {
        threadLocal.set(id);
    }

    public static Long getCurrentId() {
        return threadLocal.get();
    }

    public static void removeCurrentId() {
        threadLocal.remove();
    }
}
```

拆解：

| 部分 | 作用 |
|---|---|
| `static final ThreadLocal<Long>` | 一个全局的"柜子管理员"，管的是 Long 类型的 id |
| `setCurrentId` | 往当前线程的格子里放 id |
| `getCurrentId` | 从当前线程的格子取 id |
| `removeCurrentId` | 清空当前线程的格子（防复用残留） |

注意：三个方法都是 `static`，所以调用时直接写 `BaseContext.getCurrentId()`，不用 new 对象。

### 拦截器里会用到的两行（第 2 步要写）

```java
Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
Long empId = ((Number) claims.get("empId")).longValue();
BaseContext.setCurrentId(empId);
```

> `claims.get("empId")` 取出来是 Object，JWT 解析后数字可能是 Integer，
> 所以用 `((Number) x).longValue()` 最稳，别直接强转 Long。

---

## ⚠️ 注意事项 / 常见坑

1. **登录接口必须放行**：`registry.addInterceptor(...).excludePathPatterns("/employee/login")`
   否则登录本身被拦，你永远拿不到 token，直接死锁。
2. **必须写 remove**：不写的话，线程复用时下一个请求会读到上一个用户的 empId（串号事故）。
3. **不要把 empId 做成 static 变量**：多线程并发会互相覆盖。
4. **ThreadLocal 不是"跨线程共享"机制**：它是"每线程一份"，父子线程之间默认也不共享。
5. **改 Java 代码必须重启项目**：不重启的话你改了拦截器也没效果。

---

## 🔗 相关链接

- [第 9 课：JWT 通行证](苍穹外卖-第9课-JWT通行证.md)
- [Java 多线程](Java多线程.md)
- [Spring 基础学习](Spring基础学习.md)

## 🧩 第 2 步：LoginCheckInterceptor 逐行解析

### HandlerInterceptor 的 3 个方法

| 方法 | 执行时机 | 用途 |
|---|---|---|
| `preHandle` | Controller **之前** | 检查 token（保安查证） |
| `postHandle` | Controller 之后、返回视图前 | 本项目用不上 |
| `afterCompletion` | **整个请求结束后** | 清理 ThreadLocal（下班清柜子） |

> 接口方法都有默认空实现，只 @Override 需要的那两个即可。

### ⚠️ 实战踩坑：自动导入导错包（2026-10-08）

写这个方法时最容易犯的错：IDEA 自动补全把 `HttpServletRequest` 导成了 `java.net.http.HttpRequest`。

| 名字 | 是什么 | 症状 |
|---|---|---|
| `jakarta.servlet.http.HttpServletRequest` | ✅ 接收别人发来的请求 | 有 `getHeader()` |
| `java.net.http.HttpRequest` | ❌ JDK 的 HTTP 客户端，用来主动发请求（以后调 OpenAI API 会用） | 只有 `getHeaders()`，没有 `getHeader` |
| `java.net.http.HttpResponse` | ❌ 已经收到的响应 | 只有 `statusCode()`（读），**没有** `setStatus`（写） |

**连锁反应**：参数类型错了 → 不是重写父接口方法 → `@Override` 也标红。

> **判断技巧**：鼠标悬停在类名上看它在哪个包；`@Override` 标红第一反应就是"签名和父类对不上"。
> **口诀**：认准 `jakarta.servlet.http`。

### 五件事的顺序不能乱

```
① 判断是不是 Controller 方法     → 不是就直接放行
② 从请求头取 token
③ 验签（parseJWT）
④⑤ 验成功 → 抠出 empId → 存进 BaseContext → return true
    验失败 → 返回 401 → return false
```

**关键：必须"验成功了才存 empId，存完了才放行"。**

顺序反了（先 setCurrentId 再验签）= 把假通行证上的 id 也塞进柜子，柜子先脏了。
顺序体现的是一条**信任链**：不确定身份之前，绝不往柜子里放东西。

> 小细节：**不用**单独写 `if (token == null)`。
> token 为 null 时 `parseJWT` 直接抛异常 → 掉进 catch → 照样 401。原版就靠 try-catch 兜住。

### 完整代码（文件：interceptor/LoginCheckInterceptor.java）

```java
package com.sky.skytakeout.interceptor;

import com.sky.skytakeout.common.BaseContext;
import com.sky.skytakeout.properties.JwtProperties;
import com.sky.skytakeout.util.JwtUtil;
import io.jsonwebtoken.Claims;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class LoginCheckInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtProperties jwtProperties;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        // ① 只处理 Controller 的方法，其它资源直接放行
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }

        // ② 从请求头里取 token
        String token = request.getHeader(jwtProperties.getAdminTokenName());

        // ③④⑤ 验签 → 成功存 id 并放行；失败返回 401
        try {
            Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
            Long empId = Long.valueOf(claims.get("empId").toString());
            BaseContext.setCurrentId(empId);
            return true;
        } catch (Exception e) {
            response.setStatus(401);
            return false;
        }
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        BaseContext.removeCurrentId();
    }
}
```

### 逐块拆解

| 代码 | 为什么这么写 |
|---|---|
| `@Component` | 交给 Spring 管，才能被容器创建、才能 @Autowired 注入 |
| `implements HandlerInterceptor` | 声明"我是一个拦截器" |
| `@Autowired JwtProperties` | 密钥和请求头名字都从 yml 来 |
| `@Override` | 签名写错 IDEA 直接标红，帮你抓错 |
| `handler instanceof HandlerMethod` | 拦截器会拦到所有请求（含静态资源），只放行真正的 Controller 方法 |
| `request.getHeader(...)` | 拿请求头里的 token，名字是 yml 里配的 `token` |
| `Long.valueOf(claims.get("empId").toString())` | claims.get 返回 Object，数字可能是 Integer 或 Long，转字符串再转 Long 最保险 |
| `return true` | 放行，继续去 Controller |
| `return false` | 不放行，**请求到此终止**，Controller 根本不会被调用 |
| afterCompletion 里的 remove | 请求结束清柜子，防线程复用残留（原版视频漏了这步） |

### 拦下时怎么返回？

**方案 A（推荐，和原版一致）**

```java
response.setStatus(401);
return false;
```

- 401 = Unauthorized（未认证），官方含义就是"你没登录"
- 前端约定：看到 401 就清本地 token、跳回登录页
- 不需要返回消息体，简单标准

**方案 B：连提示语一起给前端**

```java
response.setContentType("application/json;charset=UTF-8");
response.getWriter().write("{\"code\":0,\"msg\":\"未登录\"}");
return false;
```

⚠️ **坑**：`return false` 但忘了 `setStatus(401)`，前端收到 200 却没数据，极难排查。状态码必须设对。

### ⚠️ 三个必知提醒

1. **光写拦截器不生效** —— 必须在 WebMvcConfiguration 里注册（第 3 步）
2. **注册时必须放行 /employee/login** —— 否则登录自己也被拦，永远拿不到 token
3. **jakarta vs javax** —— 原版视频是 Spring Boot 2.x 用 `javax.servlet`；
   本项目是 Spring Boot 3.2.5，必须 `jakarta.servlet.http.*`，否则编译不过
## 🧩 第 3 步：WebMvcConfiguration 注册拦截器

### 为什么还要"注册"

`@Component` 只让 Spring **创建**了这个拦截器对象，
Spring MVC 并不知道要拿它拦请求 —— 就像雇了保安但没告诉他站哪个门。

需要一个配置类来登记："这个拦截器，挂在所有请求前面。"

### 先猜：忘了放行 /employee/login 会怎样

登录接口本身也要过拦截器 → 还没登录没 token → 验签失败 → 401。
**你永远拿不到 token，因为发 token 的那扇门本身也要查验 token。** 死锁。

### 代码（文件：config/WebMvcConfiguration.java）

```java
package com.sky.skytakeout.config;

import com.sky.skytakeout.interceptor.LoginCheckInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {

    @Autowired
    private LoginCheckInterceptor loginCheckInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loginCheckInterceptor)
                .addPathPatterns("/**")
                .excludePathPatterns("/employee/login");
    }
}
```

### 逐块拆解

| 代码 | 作用 |
|---|---|
| `@Configuration` | 告诉 Spring"这是配置类，启动时请读它"；不写这个注解，Spring 根本不看这个文件 |
| `implements WebMvcConfigurer` | 往 Spring MVC 里**增补**自定义配置 |
| `@Autowired LoginCheckInterceptor` | 拿到刚才写的拦截器对象（靠它的 @Component 才注入得进来） |
| `addInterceptors(InterceptorRegistry registry)` | registry 是"拦截器登记表"，写一行生效一个 |
| `.addPathPatterns("/**")` | 拦**所有**路径 |
| `.excludePathPatterns("/employee/login")` | 放过登录接口（免检通道） |

### 路径通配符

```java
"/**"   // ✅ 所有路径，含 /employee/1 这种多层
"/*"    // ❌ 只匹配一层，/employee/1 拦不到
```

### ⚠️ 三个坑

1. **别写 `implements WebMvcConfigurationSupport`**
   网上教程常这么写，但它会把 Spring Boot 默认配置**全部顶掉**（静态资源、JSON 转换都可能坏）。
   用 `WebMvcConfigurer` 是**增补**，不是覆盖。
2. **excludePathPatterns 别拼错**
   少个 `/` 或写成 `/employee/Login` → 登录接口被拦 → **现象是登录接口返回 401**，记住这个现象好排查。
3. **现在全拦 `/**`，以后可细调**
   做 RAG 智能客服时，如果要有"游客也能访问"的接口，把那些路径加进 excludePathPatterns 即可。
## 🔑 补充精讲：这段代码到底是谁在调用（2026-10-08）

> 用户反馈"LoginCheckInterceptor 没太懂"，下面是关键补充。

### 一、preHandle 不是我调用的，是 Spring 调用的

全项目搜不到任何地方调用 `preHandle`，因为它**不需要你调**，你只是写好放在那儿，**框架在合适时机替你调**。

这叫**回调（callback）**——"你写代码，框架来调用"。Spring 里到处都是这个思路（`@RestController` 的方法也是这样被调用的）。

> 类比：你不是导演，是演员。剧本（接口）规定了第 3 幕上场、台词是什么；
> 什么时候喊你上，是导演（Spring）的事。

### 二、一次请求的完整时间线

```
① 浏览器发请求，请求头里带 token
② Tomcat 从线程池抓一个线程（T1）负责这次请求
③ DispatcherServlet 收到，开始干活
④ Spring MVC 按 URL /employee 找到 EmployeeController.findAll()
⑤ Spring MVC 想起有登记过拦截器 → 调用 preHandle(request, response, handler)
      ⚠️ 三个参数是 Spring 塞进去的，不是你传的
      ⚠️ 时机是"还没到 findAll() 之前"
⑥ 你的 preHandle 返回 true → 放行 → 调用 findAll()
                返回 false → 停止 → findAll() 一行都不会执行
⑦ 业务跑完（或被拦下）
⑧ Spring 调用 afterCompletion() → 清理
```

**⑤ 和 ⑥ 是核心**：你的代码是插在"找到方法"和"调用方法"之间的一个关卡。

### 三、三个参数是谁给的

| 参数 | 它是什么 | 类比 |
|---|---|---|
| `HttpServletRequest request` | 这次请求的全部内容（URL、请求头、请求体、参数） | 递进来的**申请表**，可以翻看 |
| `HttpServletResponse response` | 你**即将发回去**的响应，还没发出去所以能改 | 还没寄出的**回信** |
| `Object handler` | 这次请求要被谁处理（一般是 Controller 的那个方法） | 申请表上"交给哪个部门办" |

**`request.getHeader("token")` 翻译**："翻一下请求头那一栏，名叫 token 的写了什么？"

**顺带解释之前导错包的坑**：

| 类 | 语义 | 能做什么 |
|---|---|---|
| `HttpServletRequest` ✅ | **别人发给我的**请求 | 读、getHeader |
| `HttpServletResponse` ✅ | **我要发出去的**响应 | 改、setStatus |
| `java.net.http.HttpRequest` ❌ | **我自己要发出去的**请求 | 没有 getHeader |
| `java.net.http.HttpResponse` ❌ | **我已经收到的**响应 | 只有 statusCode()，改不了 |

一句话：**ServletRequest/Response 是"服务器端"的，导错的是"客户端"的。**

### 四、handler instanceof HandlerMethod 是干嘛的

因为拦截器配的是 `/**`（全拦），但有些请求根本没有 Controller 方法：

| 请求 | handler | 要验 token 吗 |
|---|---|---|
| GET /employee | HandlerMethod | ✅ |
| GET /favicon.ico | 静态资源 | ❌ |
| Swagger 页面 | 静态资源 | ❌ |

`if (!(handler instanceof HandlerMethod)) return true;`
= **"不是冲着 Controller 方法来的（静态资源），直接放行。"**

### 五、try-catch 反而是"正常流程"（重点）

`JwtUtil.parseJWT` 的设计是：**验签不通过就抛异常，不返回 null**。

| 情况 | 发生什么 | 结果 |
|---|---|---|
| 牌是真的、没过期 | 正常执行完返回 Claims | 放行 |
| 没带 token（null） | 内部抛异常 | 401 |
| 假牌 | 验签失败抛异常 | 401 |
| 牌过期 | 抛过期异常 | 401 |

**能执行到 try 的第二行，就说明验签已经通过。**

这里的 catch 不是"程序坏了"，而是"**验牌失败**"这个**预期内的正常分支**——
就像门禁刷卡失败是正常会发生的事，不是门禁坏了。

**故意不区分**"没带牌"和"假牌"：对用户结果一样（都得重新登录），区分了没用还多写代码。

### 六、return true / false 给谁看

给 Spring 看的，这是 preHandle 的"命令"：

- `return true` → 放行，继续调用 Controller
- `return false` → 拦下，**Controller / Service / Mapper 一行都不许跑**，请求到此终止

### 七、为什么 Service 里也能拿到 empId（ThreadLocal 串起来）

```
Tomcat 把这次请求交给【线程 T1】
   ↓
T1 执行 preHandle → BaseContext.setCurrentId(1L)  → 存进"T1 的柜子"
   ↓
T1 执行 EmployeeController.findAll()
T1 执行 EmployeeServiceImpl.xxx()
T1 执行 EmployeeMapper.xxx()
   ↑ 整条链路从头到尾都是同一个线程 T1
   ↓
所以 Service 里 BaseContext.getCurrentId() 能拿到 1 ✅
```

**"同一个请求的整条链路全程在同一个线程上跑"**，这就是 ThreadLocal 能当"请求上下文"的前提。

> 类比：一场婚礼从头到尾同一个司仪在跟。他把"新郎叫张三"记在自己手卡上，
> 后面每个环节翻开手卡都知道是谁；隔壁那场婚礼是另一个司仪，互不干扰。

### 八、afterCompletion 为什么是另一个方法

| 方法 | 时机 |
|---|---|
| preHandle | 进门时（Controller 之前） |
| afterCompletion | **离场时**（整个请求彻底结束） |

Java 里没有"等下结束后再执行"这种写法，所以 Spring 提供两个方法，分别在两个时间点喊你。

> 类比：preHandle = 进门刷卡，afterCompletion = 离场关灯。

afterCompletion 相当于 **finally** —— 哪怕 Controller 中间抛异常也会执行，所以清理一定做得到。

### 九、进阶细节：setCurrentId 为什么紧挨着 return true

```java
BaseContext.setCurrentId(empId);   // 先存
return true;                        // 再放行
```

| 情况 | 柜子状态 |
|---|---|
| 验签成功 → 存 id → 放行 | 柜子里正好有一个正确的 id ✅ |
| 验签失败 → catch → 拦下 | 柜子从头到尾干干净净（压根没执行到 set） ✅ |

中间**没有灰色地带**：要么"成功了且存好了"，要么"失败了且没存"。

**巧妙点**：preHandle 返回 false 时 Spring **不会**调用 afterCompletion，
但不要紧——返回 false 的那条路上根本没往柜子放东西，不需要清理。设计自洽。

### 十、整段代码一句话总结

> **每个请求进来，先翻它的请求头找 token；能验通就把它代表的员工 id 塞进当前线程的柜子，然后放行；
> 验不通就回 401 拦下。请求结束时，把柜子清空。**
## 🛡️ 补充：认证 vs 授权（2026-10-08 用户主动发现）

> 用户提问："JWT 验证通过就能 GET 所有 employee，我是 admin 才能用 GET/POST 吧？"
> **这个直觉完全正确** —— 现在的拦截器只做了"认证"，没做"授权"。

### 一、两个概念

| | 认证 Authentication | 授权 Authorization |
|---|---|---|
| 回答的问题 | **你是谁？**（有没有登录） | **你能不能做这件事？**（权限够不够） |
| 类比 | 大门保安查工牌：确认你是本公司员工 | 财务室门口查权限：你能不能进这间屋 |
| 现在做了吗 | ✅ 已做（LoginCheckInterceptor） | ❌ **完全没做** |

### 二、现在的真实状况

只要有**任意一个有效 token**（哪怕是实习生的），就能调用全部接口：

```
DELETE http://localhost:8080/employee/1
token: eyJhbGciOiJIUzI1NiJ9....
→ 200，admin 被删了 😱
```

原因：拦截器只判断"token 是真的、没过期"，**从没看这个 token 代表谁、什么级别**。

### 三、而且现在根本没法判断角色

Employee 实体里**没有 role 字段**，只有 `status`（1启用/0禁用）：

```java
private Integer status;   // ← 只有"启用/禁用"，不是角色
```

要加角色，第一步是**改数据库 + 改 Entity**：`employee 表加 role VARCHAR(20)  -- ADMIN / STAFF`

> 原版苍穹外卖视频**也没做这件事**，属于教学项目的简化。
> 真实系统这么上线，安全审计能打穿。

### 四、真要加，三种做法

| 方案 | 做法 | 难度 | 评价 |
|---|---|---|---|
| A. 字段 + 拦截器手写判断 | 表加 role，登录时塞进 JWT，拦截器里 if 判断 | 中 | 易理解，但很啰嗦 |
| B. **自定义注解 + AOP** ⭐ | `@RequireRole("ADMIN")` 标方法上，AOP 统一判断 | 中高 | **真实项目最常用、最优雅**，推荐后面学 |
| C. Spring Security | 框架全家桶 | 高 | 企业标配，学习曲线陡 |

JWT 带角色的核心两行：

```java
// 登录签发时
claims.put("empId", id);
claims.put("role", employee.getRole());

// 拦截器验完签名后取出
String role = (String) claims.get("role");
```

> ⚠️ JWT payload 是**明文可解**的：角色放进去**能被看到，但改不了**（改了就验签失败）。
> 能看到没关系，**改不了才关键**。

### 五、必须记住的一句话

> **前端把按钮藏起来 ≠ 权限控制。**

攻击者根本不用你的页面，直接用 Postman / curl 绕过前端。

**前端隐藏是"体验"，后端拦截才是"安全"。**

### 六、还有一层：数据级权限

| 术语 | 例子 |
|---|---|
| **垂直越权** | 普通员工干了管理员的事 |
| **水平越权** | A 用户看了 B 用户的数据（如查订单传别人的订单号） |

防水平越权的经典写法：

```java
// ❌ 危险：前端传谁就查谁
Result getOrders(Long userId)

// ✅ 安全：只查当前登录人
Result getOrders() {
    Long empId = BaseContext.getCurrentId();   // ← ThreadLocal 正好干这个
    ...
}
```

**`BaseContext` 就是防水平越权的基础设施** —— 这也是为什么拦截器要费劲把 empId 存柜子，而不是让前端传。

### 七、为什么现在不做

1. 主线还没完（分类/菜品/套餐），过早加权限会带偏注意力
2. 方案 B 需要 AOP，还没学
3. 纯粹的"增强"，不做不影响后端主流程

**但迟早要面对**：做 RAG 智能客服时会有"顾客不用登录也能提问"的接口，
必须区分"员工 token"和"顾客"，哪些放行哪些拦 —— **和 RAG 直接相关**。
## ❓ 常见疑问：我都有 JWT 了，为什么还要拦截器？（2026-10-08）

> 用户问："都有 JWT 验证了，为什么还要拦截器？"

### 结论先说：它们不是两件事，是**同一件事**

**JWT 验证就发生在拦截器里。** 拦截器删掉，`JwtUtil.parseJWT(...)` 就再也没人调用了。

### 项目实测（全项目搜索）

| 动作 | 出现在哪 | 备注 |
|---|---|---|
| **发牌** `createJWT` | `EmployeeServiceImpl.login()` | 全项目**唯一一处** |
| **验牌** `parseJWT` | **`LoginCheckInterceptor`** | 全项目**唯一一处** |
| **读请求头** `getHeader` | **`LoginCheckInterceptor`** | 全项目**唯一一处** |

**整个项目只有拦截器一个地方读请求头。** 删掉它，验证就凭空消失。

### 关键误解：JWT 不会自己验证自己

JWT 自带 signature，但那**不是自己显影的**。

> 类比：人民币的防伪线、水印不会自己发光告诉收银员"我是真的"——得有人拿验钞机去照。

| 角色 | 是谁 |
|---|---|
| 印钞厂 | `JwtUtil.createJWT()`（登录时生成签名） |
| **验钞机** | `JwtUtil.parseJWT()` |
| **拿机器照的人** | **`LoginCheckInterceptor`** |

不调用 `parseJWT` 的话，token 在程序眼里只是一段普通字符串——
谁都能自己捏一个 `{"empId":999,"exp":永不过期}` 塞进请求头，程序照样认，**因为没人在验**。

### 如果删掉拦截器

| 功能 | 结果 |
|---|---|
| POST /employee/login | ✅ 照样能用（发牌在 Service 里，不依赖拦截器） |
| GET /employee | ⚠️ 不管有没有 token 都照做 |
| DELETE /employee/1 | ⚠️ 同上 |

因为 **Controller 里一行读 token 的代码都没有**。删掉拦截器 → 退回第 9 课状态：

> **能登录，但登录跟没登录一模一样。**

> 💡 **破坏性小实验**（第 4 步做完后推荐做）：
> 把注册拦截器那几行临时注释掉，然后**不带 token** 发 `DELETE /employee/7` —— 居然成功了。
> 看过一次就永远忘不掉拦截器的作用。

### 为什么非得用"拦截器"这个形式

"检查登录"不属于任何一个业务（既不属于员工业务也不属于菜品业务），
但**每个业务都需要** —— 这种"到处都是的重复逻辑"就是拦截器/AOP 要解决的。这叫**横切关注点**。

| 做法 | 结果 |
|---|---|
| 每个 Controller 手写验 token | 写 28 遍，容易漏，改密钥要改 28 处 |
| 拦截器写一遍 | 一次搞定全部，**新增接口自动被保护** |

### 一张工牌串起整个故事

| 课 | 干了什么 | 类比 |
|---|---|---|
| 第 8 课 | 登录接口核对账号密码 | 去前台报名字"我是 admin" |
| 第 9 课 | 签发 JWT | 前台**印一张工牌**（名字 + 防伪标记） |
| 第 10 课 | `BaseContext` | 门口**储物柜**，存"当前进门的是谁" |
| 第 10 课 | 拦截器 | **保安上岗**，每次进楼拿刷卡机验工牌 |

> **工牌是死的塑料片，它不会自己走路去门口站岗。**
> 是保安（拦截器）拿着刷卡机（parseJWT）去验它。

### 总结

| | 提供什么 | 缺了会怎样 |
|---|---|---|
| **JWT** | **证据**（一张防伪工牌） | 保安想查，但没证可查 |
| **拦截器** | **检查的动作**（保安 + 刷卡机） | 发了证但没人查 = 等于没发 |

**证据 + 检查动作 = 安全，缺一个都白搭。**

> ⚠️ 用词纠正：第 9 课做的是"JWT **签发**"，不是"JWT 验证"。
> 验证这个动作，正是第 10 课的拦截器提供的。
## ❓ 常见疑问：拦截器"拦下来的是谁"？（2026-10-08）

### 结论：拦的不是"人"，是**这一次请求**

服务器从来不知道门外站的是谁，它只看到一个 HTTP 请求包：

```
POST /employee HTTP/1.1
Host: localhost:8080
token: eyJhbGciOiJIUzI1NiJ9...
```

拦截器能看到的**全部信息**就是这些。它不认识你，也不记得你。

> 类比：保安不认识你。他不看你脸、不记你名字，只做一件事——
> **看你手里的牌是不是真的。牌真就放，牌假/没牌就拦。**

### 具体被拦下的是哪几种请求

| 情况 | 请求头里的 token | 拦截器看到什么 | 结果 |
|---|---|---|---|
| 从来没登录过 | **压根没有**这一栏 | getHeader 返回 null | 🚫 401 |
| 自己乱编一个 | `abc123` | 解析失败 | 🚫 401 |
| 登录过但牌过期 | 真牌，时间过了 | 抛过期异常 | 🚫 401 |
| 用别的密钥签的假牌 | 格式对、签名不对 | 验签失败 | 🚫 401 |
| 正常登录、牌有效 | 真牌 + 没过期 | 验签通过 | ✅ 放行 |

**四种被拦情况在服务器眼里一模一样** —— 都是"这牌我认不了"，统一 401。
故意不区分：对用户来说结果都一样（回去重新登录）。

### 能通过的是谁

**任何持有"我们签发的、且没过期的 token"的请求。**

⚠️ 但拦截器**不关心这张牌是 admin 还是实习生的** —— 这就是"授权缺失"：

| | 现在能分辨吗 |
|---|---|
| 有没有登录 | ✅ 能 |
| 是 admin 还是实习生 | ❌ 不能（表里也没这字段） |

### 在系统眼里"人"到底是什么

```
token 的 payload = {"empId": 1, "exp": ...}
        ↓ 拦截器解出来
empId = 1L              ← 这就是"人"
        ↓
BaseContext.setCurrentId(1L)
```

**在系统里，"一个人"就是一个 Long 数字。** 1 就是 admin，7 就是 zhangsan。

### 反直觉但重要：拦截器**不记账**

拦截器没有黑名单，**什么都不记**。每次请求都从零开始验一遍牌：

| 这次 | 下次 |
|---|---|
| 没带 token → 拦下 | 带了有效 token → **照样放行** |
| 被拦不代表"记录在案" | 服务器不会"记得你上次干过坏事" |

**这就叫「无状态」（stateless）** —— 服务器每次请求都独立判断，不保存"谁登录过"。

> 这正是 JWT 相对 Session 的核心区别：
> Session 服务器记名单、能主动踢人下线；
> JWT 不记名单、只看牌 —— 所以想"强制下线"反而麻烦（得额外搞黑名单）。

### 被拦下之后，是谁收到这个消息

```
浏览器/Postman 发请求
      ↓
【拦截器】验牌失败 → response.setStatus(401) → return false
      ↓
请求到此终止：Controller 不执行、Service 不执行、数据库一动不动
      ↓
401 回到发起方
      ↓
【前端】收到 401 → 清掉本地 token → 跳回登录页
```

**`return false` 之后整个请求就断在这儿了，后面一行代码都不跑。**

### 总结

| 问题 | 答案 |
|---|---|
| 拦下来的是谁？ | 不是"人"，是**这一次请求** |
| 什么样的请求被拦？ | 拿不出有效 token 的（没带 / 假的 / 过期） |
| 服务器认识我吗？ | ❌ 不认识，只认牌 |
| 被拦了会被拉黑吗？ | ❌ 不会，不记账，下次带有效牌照样进 |
| 系统里"我"是什么？ | 一个 Long 数字（empId） |

> **拦截器不是"认人的门卫"，是"验牌的闸机"。
> 它不记你是谁，也不记你犯过错 —— 每次来，都只看这一张牌。**
## 🧩 第 3 步详解：WebMvcConfiguration 到底在干嘛（2026-10-08 补讲）

### 一、为什么还需要一个"配置类"

| 注解 | Spring Boot 的行为 |
|---|---|
| `@RestController` | 约定：自动注册成"能处理请求的类" |
| `@Service` / `@Component` | 约定：自动创建对象放进容器 |
| **拦截器** | **没有任何约定！** `@Component` 只创建了对象，但没人知道你要拿它拦请求 |

> 类比：保安已经招进公司了（对象创建好了），但**没人告诉他站哪个门**。
> 配置类就是那张"**岗位安排表**"。

**不写配置类的后果**：项目照样启动、接口照样能用、**拦截器悄悄不生效**，没有任何报错。

### 二、addInterceptors 是被 Spring 调用的（同样是回调）

| 方法 | 什么时候执行 | 执行几次 |
|---|---|---|
| `addInterceptors` | **项目启动时** | **只 1 次** |
| `preHandle` | 每个请求进来时 | **每个请求都跑** |

> 类比：`addInterceptors` = 开店前布置场地（装闸机，装一次）；
> `preHandle` = 每次有客人进门就验一次牌。

#### 启动时间线

```
① Spring 扫描包下所有类
② 看到 @Configuration → 判定"这是配置类"
③ 创建对象 new WebMvcConfiguration()
④ 看到 @Autowired LoginCheckInterceptor → 注入拦截器
⑤ Spring MVC 初始化时发现你实现了 WebMvcConfigurer
   → 调用 addInterceptors(registry)，registry 是 Spring 递过来的"登记表"
⑥ 你填表：拦 /**，放过 /employee/login
⑦ Spring MVC 记住这个安排
▶ 启动完成
```

### 三、逐行拆解

**① `@Configuration`** —— 告诉 Spring"这是配置类，请你读它"。
冷知识：`@Configuration` 自身被 `@Component` 标记，是它的**特化版**，所以扫包时能被发现。

> ⚠️ 头号坑：忘了写 → 类就是个无人理睬的普通类，**不报错、不生效、没提示**。

**② `implements WebMvcConfigurer`** —— Spring MVC 的配置接口，方法**全是 default**，只用 override 关心的那个。

| 写法 | 含义 |
|---|---|
| `implements WebMvcConfigurer` | **在默认配置上打补丁** ✅ |
| `extends WebMvcConfigurationSupport` | **从零重建整套配置**（顶掉静态资源、JSON 转换…）❌ |

**③ `@Autowired LoginCheckInterceptor`** —— 靠拦截器上的 `@Component` 才注入得进来。

> 环环相扣：拦截器写 `@Component` → 容器里有它 → 配置类才能注入它。

**④ `addInterceptors(InterceptorRegistry registry)`** —— `registry` **不是你 new 的**，是 Spring 创建好递过来的"拦截器登记表"，往里写一行就多记一个拦截器。

> ⚠️ 第二个不报错的坑：方法名写错（如少个 s）→ 变成你自己新加的方法。
> 保留 `@Override` → 会标红提醒你；删掉 `@Override` → **编译通过、启动正常、拦截器不生效**。
> **`@Override` 在这里是安全带，别删。**

**⑤ 链式调用** —— 三步，不是一句：

```java
registry.addInterceptor(loginCheckInterceptor)   // ① 登记：我要用这个拦截器
        .addPathPatterns("/**")                   // ② 拦哪些 → 返回同一个登记项
        .excludePathPatterns("/employee/login");  // ③ 放过哪些 → 返回同一个登记项
```

> 类比：填一张表——姓名栏、拦截范围栏、豁免栏。填完还是**同一张表**，所以能接着填。

拆开写完全等价：

```java
InterceptorRegistration registration = registry.addInterceptor(loginCheckInterceptor);
registration.addPathPatterns("/**");
registration.excludePathPatterns("/employee/login");
```

**⑥ `addPathPatterns("/**")`** —— 所有请求都过。
> 小知识：不写 addPathPatterns 默认也是全拦，但显式写更清楚。

**⑦ `excludePathPatterns("/employee/login")`** —— 免检通道。

**为什么必须写？**

```
想登录 → POST /employee/login → 但没 token
   → 拦截器："没牌？拦！" → 401
   → 拿不到 token → 永远登录不了 😱
```

**发牌的门本身也要查验牌照** —— 死锁。

### 四、路径通配符

| 写法 | 匹配 | 例子 |
|---|---|---|
| `/employee/login` | **精确匹配** | 只匹配这一条 |
| `/**` | **任意层级** | `/employee`、`/employee/1`、`/dish/page` 全中 ✅ |
| `/*` | **只匹配一层** | 只中 `/employee`，**不中** `/employee/1` ❌ |
| `/employee/**` | employee 下所有 | `/employee`、`/employee/1`、`/employee/login` |

**经典错误**：写成 `/*` 以为能拦全部，结果带 id 的路径全漏。
> 记住：**要全拦，写 `/**`。**

### 五、启动 + 请求 全景图

```
【启动时，只跑一次】
  Spring 扫描 → 发现 @Configuration → 创建 WebMvcConfiguration
    → 注入 LoginCheckInterceptor
    → 调用 addInterceptors(registry)
    → 登记：拦 /**，放过 /employee/login
  ✅ 拦截器正式上岗

【之后每个请求，都跑一遍】
  请求进来
    → 命中豁免清单？(/employee/login) → 直接放行
    → 没命中 → 调用 preHandle
         验牌成功 → setCurrentId → 放行 → Controller → Service → Mapper
         验牌失败 → 401 → 拦下
    → 请求结束 → afterCompletion → 清柜子
```

### 六、本步四个坑

| # | 坑 | 现象 | 正解 |
|---|---|---|---|
| 1 | 忘写 `@Configuration` | **不报错！** 拦截器悄悄不生效 | 必须加 |
| 2 | 方法名/参数写错 + 删了 `@Override` | **不报错**，拦截器不生效 | 保留 `@Override` |
| 3 | excludePathPatterns 路径拼错 | 登录接口返回 **401** | 必须 `/employee/login`，别多写末尾斜杠 |
| 4 | 用了 `extends WebMvcConfigurationSupport` | 静态资源、JSON 转换可能坏 | 用 `implements WebMvcConfigurer` |

**第 1、2 条最阴险——它们不报错**，所以一定要做第 4 步的实测。
## 📋 一张表说清：WebMvcConfiguration 到底起什么作用（2026-10-08）

> 一句话：**它自己一张牌都不验，只负责"把保安安排到门口，并告诉他哪些人能免检"。**
> 它里面**没有一行代码跟 token 有关**。

### 它解决的是"知道"和"用上"之间的鸿沟

```
① LoginCheckInterceptor 写了 @Component
      → Spring 知道"有这么一个对象"          ← 知道
② 但 Spring MVC 不知道"要不要拿它拦请求"     ← 中间断了！
③ WebMvcConfiguration 里 addInterceptors(...)
      → Spring MVC 收到"好，挂在 /** 上"      ← 用上
```

**第 ② 步就是断层，配置类是架在上面的桥。**
> 没有它：保安站在公司里，但没人告诉他站哪个门。

### 它独有的权力：决定"查哪里"

翻一遍 `LoginCheckInterceptor` —— 里面**一个字都没提** `/employee`，也没提放过哪些路径。
它只写了"**怎么查牌**"：

```java
String token = request.getHeader(...);   // 怎么取牌
JwtUtil.parseJWT(...);                    // 怎么验牌
BaseContext.setCurrentId(empId);          // 验完存哪
```

而"**查哪里**"只存在于配置类里：

```java
.addPathPatterns("/**")                    // 守哪里：全部
.excludePathPatterns("/employee/login");   // 免检通道：登录
```

**关注点分离：**

| | 关心什么 |
|---|---|
| `LoginCheckInterceptor` | **怎么查**（验牌的动作） |
| `WebMvcConfiguration` | **查哪里**（适用范围） |

> 类比：螺丝刀（拦截器）不知道自己要去拧哪个螺丝；**装配说明书（配置类）**才知道。

**好处**：要放开某个接口给游客，**只改配置类一行**，拦截器一个字不用动；
要换验牌方式，只改拦截器，路径规则不受影响。

### 三个角色的分工表（背下来）

| 类 | 它是什么 | 负责什么 | 什么时候跑 |
|---|---|---|---|
| `LoginCheckInterceptor` | 🔒 **保安** | **怎么查牌**（验 token 的逻辑） | 每个请求都跑 |
| `WebMvcConfiguration` | 📋 **岗位安排表** | **查哪里**（哪些路径查、哪些免检） | **启动时只跑 1 次** |
| `BaseContext` | 🗄️ **储物柜** | **存查到的结果**（empId） | 每次 set / get |

```
请求 GET /employee?token=xxx
   ↓
📋 岗位安排表（启动时已定好）回答："这个路径要查！"
   ↓
🔒 保安动手：取牌 → 验牌 → 通过
   ↓
🗄️ 把 empId 放进柜子
   ↓
放行 → Controller → Service（Service 里能 getCurrentId() 拿到柜子里的值）
```

### 为什么不能把路径写死在拦截器里

```java
// ❌ 能跑，但很糟糕
if (request.getRequestURI().equals("/employee/login")) {
    return true;
}
```

| 坏处 | 说明 |
|---|---|
| **规则散了** | 想看"哪些接口受保护"，得翻拦截器代码，还可能散在好几个 if 里 |
| **拦截器不通用** | 被绑死在"employee 登录"上，以后做顾客端还得复制一份改 |

现在的写法把规则**集中在一处，一眼看清**。

### 三行代码 = 岗位安排表的三个栏目

```java
registry.addInterceptor(loginCheckInterceptor)   // ① 派谁去   → LoginCheckInterceptor
        .addPathPatterns("/**")                   // ② 守哪里   → 全部路径
        .excludePathPatterns("/employee/login");  // ③ 免检通道 → 登录接口
```

### 删掉它会怎样

| 删掉后 | 结果 |
|---|---|
| 项目能启动吗 | ✅ 能，**不报错** |
| 登录能用吗 | ✅ 能（发牌在 Service 里，不依赖拦截器） |
| 拦截器工作吗 | ❌ **完全瘫痪** |
| 不带 token 能查员工吗 | 😱 **能，200** |

> **它是"拦截器"和"请求"之间唯一的那根线。线断了，拦截器就是一段没人调用的死代码。**
## ✅ 第 10 课通关（2026-10-08）

三个测试全部通过：

| 测试 | 结果 | 证明了什么 |
|---|---|---|
| ① POST /employee/login | ✅ 200 | 免检通道生效 |
| ② GET /employee（不带 token） | 🚫 401 | 拦截器上岗了 |
| ③ GET /employee（带 token） | ✅ 200 | 验牌通过能放行，empId 已进柜子 |

**本课最终成果**：登录发 token → 每次请求自动验 token → 未登录 401 → 登录人 empId 存进 ThreadLocal → 请求结束自动清理。

---

## 📌 下一课预告：这些代码什么时候真正派上用场

| 代码 | 什么时候用上 |
|---|---|
| `BaseContext.getCurrentId()` | 第 11 课分类管理开始，用于填充 `create_user` / `update_user` |
| 公共字段自动填充 | 分类表、菜品表、套餐表**都有** create_time/create_user/... → 抽出来统一处理（**要用 AOP**） |
---

*创建于 2026-10-08*