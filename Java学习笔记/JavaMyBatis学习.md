# Java MyBatis 学习笔记（课程版）

> 📅 开课日期：2026-09-26
> 🛠️ 工具：IDEA + Maven + MySQL + Navicat
> 🗄️ 数据库：JDBCTest

---

## 📖 课程目录

- [x] 第 1 课：MyBatis 是什么 & 为什么需要它
- [x] 第 2 课：配置 MyBatis 并跑通第一个查询 ✅
- [ ] 第 3 课：
- [ ] 第 4 课：
- [ ] 第 5 课：

---

## 第 1 课：MyBatis 是什么 & 为什么需要它

### JDBC 的痛点
- 每次都要手动：获取连接 → 设参数 → 执行 → 手动封装 ResultSet
- 大量重复模板代码，MyBatis 帮你自动做这些重复工作

### MyBatis 解决了什么
| JDBC | MyBatis |
|------|---------|
| 手动管理连接 | 自动管理 |
| 手动设参数 | 自动映射参数 |
| 手动封装结果集 | 自动把行转成 Java 对象 |
| 大量模板代码 | 你只需写 SQL 和接口 |

### 核心思想
```
你写接口 + 你写 SQL  →  MyBatis 自动生成实现，帮你调 JDBC
```

---

## 第 2 课：配置 MyBatis 并跑通第一个查询

### 项目结构
```
src/main/java/org/example/MyBatis/
├── StudentMapper.java    ← 接口（方法签名）
├── MyBatisUtil.java      ← 工具类
└── MyBatisTest.java      ← 测试类

src/main/resources/
├── mybatis-config.xml              ← 核心配置
└── org/example/MyBatis/
    └── StudentMapper.xml           ← SQL 映射
```

### 4 个关键文件（理解即可，不用背）

### ① mybatis-config.xml
配数据库连接信息 + 注册 Mapper XML
```xml
<dataSource type="POOLED">
    <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/JDBCTest?..."/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</dataSource>
<mapper resource="org/example/MyBatis/StudentMapper.xml"/>
```

### ② StudentMapper.java（接口）
```java
public interface StudentMapper {
    Student findById(int id);
}
```

### ③ StudentMapper.xml（SQL）
```xml
<mapper namespace="org.example.MyBatis.StudentMapper">
    <select id="findById" resultType="org.example.MyJDBC.Student">
        SELECT * FROM student WHERE id = #{id}
    </select>
</mapper>
```
- `namespace` = 接口的全限定名
- `id` = 接口的方法名
- `#{id}` = 参数占位符

### ④ MyBatisUtil.java（工具类）
```java
SqlSessionFactory factory = new SqlSessionFactoryBuilder()
        .build(Resources.getResourceAsStream("mybatis-config.xml"));
SqlSession session = factory.openSession(true);  // true = 自动提交
```

### 使用
```java
try (SqlSession session = MyBatisUtil.getSqlSession()) {
    StudentMapper mapper = session.getMapper(StudentMapper.class);
    Student s = mapper.findById(1);
    System.out.println(s);
}
```

---

