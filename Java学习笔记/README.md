# Java 学习笔记 📚

> 📦 **GitHub**: [github.com/yukinoacb-hash/-](https://github.com/yukinoacb-hash/-)
> 🗂️ 按主题分文件夹存放，下面是总索引。

---

## 📋 笔记索引

### ☕ Java 基础(`Java基础/`)
- [📄 方法引用](Java基础/Java方法引用.md) — `::` 语法糖，让 Lambda 更简洁
- [📄 异常处理](Java基础/Java异常处理.md) — try-catch / throws / throw
- [📄 多线程](Java基础/Java多线程.md) — 线程创建、线程安全、线程池、Callable
- [📄 反射与注解](Java基础/Java反射与注解.md) — Class.forName、invoke、动态调用
- **IO 流**
  - [📄 File 类](Java基础/JavaFile类.md) — 文件和文件夹路径的操作
  - [📄 IO 流（详解）](Java基础/JavaIO流.md) — 字节流、字符流、缓冲流、转换流、对象流、打印流
  - [📄 IO 流（速记）](Java基础/Java%20IO流.md) — 输入/输出 + 两条主线的极简版
  - [📄 NIO](Java基础/JavaNIO.md) — Path、Files 工具类（新 IO）

### 🗄️ 数据库（JDBC）(`JDBC/`)
- [📄 JDBC](JDBC/JavaJDBC.md) — 连接数据库、PreparedStatement、事务、连接池
- [📄 JDBC（课程版）](JDBC/JavaJDBC学习.md) — 跟着课程走的 JDBC 分课记录
- [📄 JDBC 实战](JDBC/JavaJDBC实战.md) — 学生管理系统（三层架构 + 事务）

### 🧩 MyBatis(`MyBatis/`)
- [📄 MyBatis 学习](MyBatis/JavaMyBatis学习.md) — 为什么用 MyBatis、配置文件、第一个查询、注解 vs XML

### 🌱 Spring(`Spring/`)
- [📄 Spring 基础](Spring/Spring基础学习.md) — IoC / DI、Spring 容器与 Bean

### 🥡 苍穹外卖项目实战(`苍穹外卖/`)
- [🗺️ 学习进度与路线图](苍穹外卖/苍穹外卖-学习进度与路线图.md) — 已完成 / 进行中 / 待学清单 + 已踩过的坑总表
- [📄 第 1 课：三层架构](苍穹外卖/苍穹外卖-第1课-三层架构.md) — Spring Boot 项目搭建 + Controller/Service/Mapper
- [📄 第 2 课：根据 id 查询](苍穹外卖/苍穹外卖-第2课-根据id查询.md) — @PathVariable、#{} 占位符、Result<Employee>
- [📄 第 3 课：新增员工 POST](苍穹外卖/苍穹外卖-第3课-新增员工POST.md) — @PostMapping、@RequestBody、405 排查
- [📄 第 4 课：删除员工 DELETE](苍穹外卖/苍穹外卖-第4课-删除员工DELETE.md) — @DeleteMapping、@PathVariable、RESTful 方法对照表
- [📄 第 5 课：修改员工 PUT](苍穹外卖/苍穹外卖-第5课-修改员工PUT.md) — @PutMapping、@Update、UPDATE 忘写 WHERE 的事故
- [📄 第 6 课：动态 SQL（MyBatis XML）](苍穹外卖/苍穹外卖-第6课-动态SQL%28MyBatis%20XML%29.md) — <if> / <set>、namespace 绑定规则、全字段更新的正解
- [📄 第 7 课：登录与密码加密（MD5）](苍穹外卖/苍穹外卖-第7课-登录与密码加密%28MD5%29.md) — 为什么不能明文存密码、MD5 三大特点、彩虹表/加盐/BCrypt、DigestUtils
- [📄 第 8 课：登录（上）—— 验身份 + DTO](苍穹外卖/苍穹外卖-第8课-登录与DTO.md) — 登录拆成两步、DTO/Entity 的区别、登录数据流
- [📄 第 9 课：JWT 通行证](苍穹外卖/苍穹外卖-第9课-JWT通行证.md) — 无状态问题、Session vs JWT、三段结构、为什么伪造不了、jjwt 依赖
- [📄 第 10 课：拦截器 + ThreadLocal](苍穹外卖/苍穹外卖-第10课-拦截器与ThreadLocal.md) — 谁在查通行证、为什么用 ThreadLocal、多线程为什么要每线程一份、remove 的坑
- [📄 第 11 课：分类管理（Category）](苍穹外卖/苍穹外卖-第11课-分类管理.md) — 5 个接口拆解、category 表设计、新增 + 分页查询（含实测验收）
- [🔁 复习速查：登录与 JWT](苍穹外卖/苍穹外卖-复习-登录与JWT速查.md) — 第 7~9 课一页速查 + 自测 10 题（含答案折叠）
- [🔗 全链路复习：从 MD5 到拦截器注册](苍穹外卖/苍穹外卖-复习-MD5到拦截器全链路.md) — 一条线串起第 7~10 课 + 分站版 + 自测题
- [📄 阶段复习：员工 CRUD 总结](苍穹外卖/苍穹外卖-阶段复习-员工CRUD总结.md) — 知识地图 + 自测 12 问 + 项目现状快照

### 🧮 力扣热题 100(`力扣热题100/`)
- [📄 第 1 题：两数之和（Two Sum）](力扣热题100/001-两数之和.md) — 哈希表入门，用空间换时间，O(n²) → O(n)

### 🛠️ 工具与规范(`工具与规范/`)
- [📄 Git 学习笔记](工具与规范/Git学习笔记.md) — 仓库、提交、远程推送

### 🔁 综合复习(`综合复习/`)
- [📄 全栈复习（JDBC → MyBatis → Spring Boot）](综合复习/复习笔记.md) — 一条线串起数据库访问三件套
- [📄 软件测试实验一：三个经典被测程序](综合复习/软件测试实验一-三个经典被测程序.md) — 三角形/NextDate/佣金问题 + 边界值、缺陷注入与测试用例

---

*最后更新: 2026-10-14*