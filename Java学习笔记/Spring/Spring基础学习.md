# Spring 基础学习笔记（课程版）

> 📅 开课日期：2026-09-26
> 🛠️ 框架：Spring Boot 4 + Spring Framework 7

---

## 📖 课程目录

- [x] 第 1 课：Spring 是什么 & 为什么需要它
- [x] 第 2 课：IoC 和 DI（控制反转 & 依赖注入）
- [x] 第 3 课：Spring 容器和 Bean
- [x] 第 4 课：各种注解的区别
- [x] 第 5 课：@SpringBootApplication 做了什么
- [x] 第 6 课：@Transactional — Spring 的事务管理
- [x] 第 7 课：REST API（HTTP 4 种请求 + 3 种取参注解）
- [x] 第 8 课：MyBatis 补充知识
- [x] 第 9 课：Spring MVC 是什么（第 10 课前置）

---

## 第 1 课：Spring 是什么 & 为什么需要它

### 传统 Java 的问题
```java
public class StudentService {
    private StudentDAO dao = new StudentDAO();  // 自己 new
}
```
- 每个类都要自己 new 依赖的对象
- 要换实现类时，所有用到的地方都要改
- 对象多了以后，创建、管理、销毁很麻烦

### Spring 的做法
```java
@Autowired
private StudentDAO dao;  // 不用 new，Spring 自动给
```
- 对象不用你 new 了，Spring 帮你管
- 想换实现类，改一处配置就行
- 对象默认单例（一个类只创建一个对象），省内存

### 类比
- 没有 Spring：你亲自买菜、洗菜、切菜、炒菜
- 有 Spring：你点菜，厨师（Spring）做好端上来

---

## 第 2 课：IoC 和 DI

### IoC（控制反转 Inversion of Control）
- 传统：你自己 `new Xxx()` → 你控制对象的创建
- Spring：你不用 new，Spring 创建好给你 → 控制权"反转"给了 Spring

### DI（依赖注入 Dependency Injection）
- `@Autowired` = Spring 把你需要的对象"注入"到你的代码里
- 你依赖什么，Spring 就给你注入什么

### 一句话
```
IoC = 对象不用你 new 了，Spring 帮你管
DI  = 你需要的对象，Spring 直接送给你（@Autowired）
IoC 是目标，DI 是手段
```

---

## 第 3 课：Spring 容器和 Bean

### 容器（IoC Container）
Spring 启动时创建一个"大箱子"，里面放着所有被管理的对象。

### Bean
被 Spring 管理的对象就叫 Bean。

### Spring 启动流程
```
@SpringBootApplication 扫描当前包
    ↓
看到 @Mapper、@RestController、@Service
    ↓
为每个类创建对象（Bean）
    ↓
放进容器里
    ↓
谁要用 @Autowired → 从容器里拿出来给他
```

---

## 第 4 课：各种注解的区别

### 按层分

| 注解 | 层 | 作用 |
|------|----|------|
| `@RestController` | 表现层 | 接收 HTTP 请求，返回 JSON |
| `@Service` | 业务层 | 写业务逻辑 |
| `@Mapper` | 数据层 | MyBatis 的 Mapper，操作数据库 |
| `@Autowired` | 任意 | Spring 自动注入对象 |

### 本质
`@RestController`、`@Service`、`@Mapper` 全都是 `@Component` 的变种，加了它们 Spring 就会把类注册成 Bean。加不同的名字只是为了**区分层次**。

---

## 第 5 课：@SpringBootApplication

做了 3 件事：
```java
@SpringBootApplication
    ├── @ComponentScan          → 扫描当前包，找 Bean
    ├── @EnableAutoConfiguration → 自动配置（Tomcat、MyBatis、DataSource...）
    └── @SpringBootConfiguration → 标记这是配置类
```

### @ComponentScan
扫描 `org.example.springbootmybatis` 及所有子包，找到带注解的类注册成 Bean。

### @EnableAutoConfiguration
你只加了 `mybatis-spring-boot-starter` 依赖，它就自动帮你配好了：
- DataSource（连接池）
- SqlSessionFactory
- 内嵌 Tomcat

---

## 第 6 课：@Transactional

### 解决的问题
多条 SQL 要么全成功，要么全回滚（如转账：A 扣钱 + B 加钱）。

### 用法
```java
@Transactional  // 加这一行就够了！
public void transfer() {
    mapper.update(fromId, -100);  // 扣钱
    mapper.update(toId, +100);    // 加钱
    // 报错 → 自动回滚，钱不会丢
}
```

### 原理
```
@Transactional
    ↓
Spring 拦截这个方法
    ↓
先开启事务 → 执行你的代码
    ├── 没报错 → commit（提交）
    └── 报错了 → rollback（回滚）
```

### 能力是数据库的，方便是 Spring 给的
```java
// JDBC 手动事务
conn.setAutoCommit(false);
try { conn.commit(); }
catch { conn.rollback(); }

// Spring @Transactional — 一行搞定
@Transactional
```

---

## 第 7 课：REST API

### HTTP 4 种请求方式

| 操作 | HTTP 方式 | Controller 注解 |
|------|----------|----------------|
| 查 | GET | `@GetMapping` |
| 增 | POST | `@PostMapping` |
| 改 | PUT | `@PutMapping` |
| 删 | DELETE | `@DeleteMapping` |

### 3 种取参注解

| 注解 | 从哪拿数据 | 示例 URL |
|------|-----------|---------|
| `@PathVariable` | URL 路径里 | `/students/{id}` |
| `@RequestParam` | URL 问号后面 | `/search?name=张` |
| `@RequestBody` | 请求体（JSON 转 Java 对象） | POST 请求体 `{"name":"张"}` |

### REST API 设计规范
```
GET    /students         查全部
GET    /students/1       查单个
POST   /students         新增
PUT    /students/1       修改
DELETE /students/1       删除
GET    /students/search  搜索
```

### 三层架构
```
Controller → Service → Mapper
  调          调        执行 SQL
```
方向是单向的，不能反过来。

---

## 第 8 课：MyBatis 补充知识

### MyBatisX 插件
- 装好后，接口方法旁边有小箭头，点击跳到 XML 对应位置
- Alt+Enter 自动生成 XML SQL 标签骨架

### 动态 SQL（可选参数）
```xml
<select id="findByCondition" resultType="...">
    SELECT * FROM student
    WHERE 1=1
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name}, '%')
    </if>
    <if test="age != null">
        AND age &lt; #{age}
    </if>
</select>
```

---

## 第 9 课：Spring MVC 是什么（第 10 课前置知识）

### 一句话

Spring MVC 是 **Spring 家族里专门"处理网页请求"的模块** —— 请求怎么找到你那个方法、参数怎么变成对象、结果怎么变回 JSON，全是它干的。

### 拆开念：MVC

| 字母 | 全称 | 含义 | 在你项目里是谁 |
|---|---|---|---|
| M | Model | 数据 | `Employee` / `Result` 里的 `data` |
| V | View | 页面 | 前后端分离后基本不用（你返回的是 JSON，不是页面） |
| C | Controller | 控制器 | `EmployeeController` |

### 一个请求进来，Spring MVC 干了这些（对着你的代码看）

```
POST /employee/login  {username, password}
   │
   ▼
① DispatcherServlet       总调度台，所有请求先到它手里
   │  ② 查"路由表"：URL=/employee/login 且是 POST → 该调哪个方法？
   │        → EmployeeController.login()
   ▼
③ 拦截器（马上要写的"保安"）   ← 第 10 课就在这里插一脚
   │
   ▼
④ 请求体 JSON 转成对象      {"username":...} → EmployeeLoginDTO（@RequestBody 的功劳）
   │
   ▼
⑤ 调用你的 login(dto)
   │
   ▼
⑥ 返回值 Result 转成 JSON   （@ResponseBody 的功劳）
   │
   ▼
⑦ 写回浏览器
```

### Spring / Spring MVC / Spring Boot 的关系（面试常问）

```
Spring Boot（脚手架 + 自动配置）
   │  帮你把下面这些"一键装好"
   ├── Spring 内核（IoC 容器 / DI / AOP）    ← 第 1~4 课学的 Bean 管理
   └── Spring MVC（Web 层：收请求、发响应）   ← 你写的 Controller 全靠它
```

| 名字 | 是什么 | 管什么 |
|---|---|---|
| **Spring** | 框架内核 | 管对象（Bean）、依赖注入、事务 |
| **Spring MVC** | Spring 的一个模块 | 管"HTTP 请求 → 你的方法 → HTTP 响应" |
| **Spring Boot** | 脚手架 / 自动配置（严格说不是框架） | 把上面这些都装好、配好，你只写业务 |

### 你其实一直在用它 —— 这些注解全是 Spring MVC 的

| 注解 | 作用 |
|---|---|
| `@RestController` | 声明这是个接收请求的类（= `@Controller` + `@ResponseBody`） |
| `@RequestMapping` / `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` | 把 URL 绑到方法上（路由表） |
| `@RequestBody` | 请求体 JSON → 对象 |
| `@PathVariable` | URL 里的 `{id}` → 方法参数 |
| `@RequestParam` | URL 里的 `?key=value` → 方法参数 |

> 所以三层架构里，**Controller 那一层就是 Spring MVC 的地盘**；Service / Mapper 跟它没关系。

### 为什么第 10 课要讲它

因为**拦截器就是 Spring MVC 提供的机制** —— 它插在"DispatcherServlet 之后、Controller 之前"那一环。
理解了上面这张流程图，就知道"保安"该站哪个位置了。