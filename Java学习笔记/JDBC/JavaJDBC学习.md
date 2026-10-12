# Java JDBC 学习笔记（课程版）

> 📅 开课日期：2026-09-25
> 📦 包名：org.example.MyJDBC
> 🛠️ 工具：IDEA + Maven + MySQL + Navicat
> 🗄️ 数据库：JDBCTest

---

## 📖 课程目录

- [x] 第 1 课：JDBC 是什么 & 6 步流程
- [x] 第 2 课：第一个 JDBC 程序（连接成功 ✅）
- [x] 第 3 课：PreparedStatement 查数据（SELECT）
- [x] 第 4 课：增删改（INSERT / UPDATE / DELETE）
- [x] 第 5 课：事务管理
- [x] 第 6 课：连接池 HikariCP

---

## 第 1 课：JDBC 是什么

### JDBC 本质
- Java 官方在 `java.sql` 包中定义的**数据库操作接口**
- **接口 Java 写，实现厂商做** → 换数据库只需换驱动，代码不变
- 类比：JDBC = USB 标准，MySQL 驱动 = 数据线

### JDBC 核心接口
| 接口 | 作用 |
|------|------|
| `DriverManager` | 管理驱动，获取连接 |
| `Connection` | 跟数据库的一个连接 |
| `PreparedStatement` | 预编译的 SQL 语句 |
| `ResultSet` | 查询结果集 |

### JDBC 6 步（背）
```
① 获取连接     → DriverManager.getConnection(url, user, pwd)
② 创建语句     → conn.prepareStatement(sql)
③ 设置参数     → pstmt.setInt(1, 20)
④ 执行 SQL     → pstmt.executeQuery() / executeUpdate()
⑤ 处理结果     → rs.next() / rs.getInt("列名")
⑥ 关闭资源     → try-with-resources 自动关闭
```

---

## 第 2 课：第一个 JDBC 程序

### 连接 URL 结构
```
jdbc:mysql://主机:端口/数据库名?参数
jdbc:mysql://localhost:3306/JDBCTest?useSSL=false&serverTimezone=Asia/Shanghai
```
- localhost = 数据库在本机
- 3306 = MySQL 默认端口
- JDBCTest = 你的数据库名

### 连接代码
```java
String url = "jdbc:mysql://localhost:3306/JDBCTest"
        + "?useSSL=false&serverTimezone=Asia/Shanghai";
try (Connection conn = DriverManager.getConnection(url, "root", "密码")) {
    System.out.println("✅ 连接成功！");
}
```

### try-with-resources
```java
try (资源) { ... }  // 大括号结束自动 close()，不用写 finally
```

---

## 第 3 课：PreparedStatement 查询（SELECT）

### 什么是 PreparedStatement？
- SQL 用 `?` 占位符，值和 SQL 分开传
- 先预编译，后传参 → **防 SQL 注入 + 性能好**

### 代码模板
```java
String sql = "SELECT * FROM student WHERE age > ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setInt(1, 20);
ResultSet rs = pstmt.executeQuery();

while (rs.next()) {
    int id = rs.getInt("id");
    String name = rs.getString("name");
    int age = rs.getInt("age");
}
```

### ResultSet 常用方法
| 方法 | 说明 |
|------|------|
| `rs.next()` | 移到下一行，有数据返回 true |
| `rs.getInt("列名")` | 取 int 类型 |
| `rs.getString("列名")` | 取 String 类型 |

---

## 第 4 课：增删改（INSERT / UPDATE / DELETE）

### 核心区别
| 方法 | 用途 | 返回值 |
|------|------|--------|
| `executeQuery()` | SELECT | `ResultSet` |
| `executeUpdate()` | INSERT / UPDATE / DELETE | `int`（受影响行数） |

### INSERT — 增
```java
String sql = "INSERT INTO student(name, age) VALUES(?, ?)";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "赵六");
pstmt.setInt(2, 21);
int rows = pstmt.executeUpdate();
```
- `VALUES(?, ?)` 的 `?` 对应列的顺序，第 1 个 ? = 第 1 列的值，以此类推
- `setString(1, "赵六")` 的 `1` 表示第 1 个 `?`，从 1 开始数

### UPDATE — 改
```java
String sql = "UPDATE student SET age = ? WHERE id = ?";
pstmt.setInt(1, 25);     // 新年龄
pstmt.setInt(2, 1);      // 定位 id=1
int rows = pstmt.executeUpdate();
```

### DELETE — 删
```java
String sql = "DELETE FROM student WHERE id = ?";
pstmt.setInt(1, 1);
int rows = pstmt.executeUpdate();
```

### ⚠️ 重要原则
- **定位靠 id（主键）**，每一行都有唯一编号
- UPDATE 和 DELETE **必须先写 WHERE，再回头补条件**
- `WHERE` 是条件，不是 `WHILE`
- `executeUpdate()` 返回受影响的行数，0 表示没找到匹配的数据

---

## 第 5 课：事务管理

### 为什么需要事务？
多条 SQL 要么全部成功，要么全部回滚（如转账：A 扣钱 + B 加钱）

### 事务模板（背）
```java
Connection conn = null;
try {
    conn = DriverManager.getConnection(...);
    conn.setAutoCommit(false);    // ① 关自动提交

    // ... 执行多条 SQL ...

    conn.commit();                 // ② 全部成功 → 提交
} catch (Exception e) {
    if (conn != null) conn.rollback();  // ③ 出错 → 回滚
} finally {
    if (conn != null) {
        conn.setAutoCommit(true);  // ④ 恢复自动提交
        conn.close();
    }
}
```

---

## 第 6 课：连接池 HikariCP

### 为什么需要连接池？
- 每次操作都 `getConnection()` → 用完 `close()` → **反复创建销毁，很慢**
- 连接池：启动时预先创建一批连接放在池里，用的时候借，用完归还 → **复用，快**

### Maven 依赖
```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

### DBUtil 工具类
```java
package org.example.MyJDBC;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

public class DBUtil {
    private static final DataSource DATA_SOURCE;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/JDBCTest"
                + "?useSSL=false&serverTimezone=Asia/Shanghai");
        config.setUsername("root");
        config.setPassword("123456");
        config.setMaximumPoolSize(10);
        DATA_SOURCE = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return DATA_SOURCE.getConnection();
    }
}
```

### 使用（对比）
```java
// 旧：每次手动 DriverManager.getConnection(...)
try (Connection conn = DriverManager.getConnection(...)) { ... }

// 新：从连接池借
try (Connection conn = DBUtil.getConnection()) { ... }
// conn.close() 是归还连接，不是真正关闭
```

---

## 最终 pom.xml（JDBC 完整版）

```xml
<dependencies>
    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.0.33</version>
    </dependency>

    <!-- HikariCP 连接池 -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
    </dependency>
</dependencies>
```
