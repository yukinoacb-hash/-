# Java JDBC — 数据库连接

> **JDBC（Java Database Connectivity）** 是 Java 官方提供的数据库操作接口（API），位于 `java.sql` 和 `javax.sql` 包中。  
> **接口由 Java 定义，实现由数据库厂商提供（驱动 jar 包）**，同一套 API 可切换不同数据库。

---

## 1. JDBC 核心架构

```
Java 程序  →  JDBC API（接口）  →  数据库驱动  →  数据库（MySQL/Oracle/...）
```

| 角色 | 说明 |
|------|------|
| **JDBC API** | Java 标准接口，定义连接、执行 SQL、处理结果等规范 |
| **数据库驱动** | 厂商实现的 jar 包，例如 `mysql-connector-j` |
| **应用程序** | 只依赖 JDBC 接口，不直接依赖厂商实现 |

---

## 2. 连接数据库（MySQL）

### 2.1 Maven 依赖

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.4.0</version>
</dependency>
```

### 2.2 获取连接

```java
package org.example.MyJDBC;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class JDBCConnection {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/java_test"
                + "?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8";
        String user = "root";
        String password = "your_password";

        try (Connection conn = DriverManager.getConnection(url, user, password)) {
            System.out.println("数据库连接成功！" + conn);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

> **try-with-resources**（Java 7+）：`try (资源)` 会自动调用 `close()`，省去 finally 手动关闭。

---

## 3. Statement — 执行 SQL

> Statement 用于执行静态 SQL，存在 SQL 注入风险，不推荐实际使用。

### 3.1 查询

```java
Statement stmt = conn.createStatement();
String sql = "SELECT * FROM student";
ResultSet rs = stmt.executeQuery(sql);

while (rs.next()) {
    int id = rs.getInt("id");
    String name = rs.getString("name");
    int age = rs.getInt("age");
    System.out.println(id + " - " + name + " - " + age);
}
```

### 3.2 SQL 注入演示

```
输入 name = "张三' OR '1'='1"
拼接 SQL：SELECT * FROM student WHERE name = '张三' OR '1'='1'
结果：返回所有学生数据！
```

---

## 4. PreparedStatement — 防 SQL 注入（重点）

### 4.1 核心优势

| 特性 | 说明 |
|------|------|
| **预编译** | SQL 先发给数据库编译，参数后传，值和 SQL 分离 |
| **防注入** | 参数中的特殊字符会被自动转义 |
| **性能好** | 同一条 SQL 多次执行时，只需传参数，不必重新编译 |
| **可读性** | 使用 `?` 占位符，代码更清晰 |

### 4.2 基本用法

```java
String sql = "SELECT * FROM student WHERE name = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "张三");
ResultSet rs = pstmt.executeQuery();
while (rs.next()) {
    System.out.println(rs.getInt("id") + " - " + rs.getString("name"));
}
```

### 4.3 增删改

```java
String sql = "INSERT INTO student(name, age) VALUES(?, ?)";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, "李四");
pstmt.setInt(2, 22);
int rows = pstmt.executeUpdate();
```

---

## 5. 事务管理

### 5.1 为什么需要事务？

**转账场景：A 扣 1000 → B 加 1000**  
如果 A 扣完钱、给 B 加钱之前程序崩溃了，钱就丢了。  
事务保证：要么全部成功，要么全部回滚。

### 5.2 JDBC 事务操作

```java
conn.setAutoCommit(false);  // 关闭自动提交

try {
    // 执行多条 SQL...
    conn.commit();           // 全部成功 → 提交
} catch (Exception e) {
    conn.rollback();         // 出错 → 全部回滚
} finally {
    conn.setAutoCommit(true); // 恢复自动提交
    conn.close();
}
```

---

## 6. 连接池（DataSource）

### 6.1 为什么需要连接池？

无连接池：每次请求创建连接 → 执行 SQL → 关闭连接（反复创建销毁，开销大）
有连接池：启动时预先创建一批连接放在池中，借连接 → 执行 SQL → 归还连接（复用）

### 6.2 HikariCP（推荐）

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/java_test"
        + "?useSSL=false&serverTimezone=Asia/Shanghai");
config.setUsername("root");
config.setPassword("your_password");
config.setMaximumPoolSize(10);

DataSource ds = new HikariDataSource(config);
Connection conn = ds.getConnection();  // 从池中借
// conn.close() → 归还到池中（不是真正关闭）
```

### 6.3 封装工具类

```java
public class DBUtil {
    private static final DataSource dataSource;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/java_test"
                + "?useSSL=false&serverTimezone=Asia/Shanghai");
        config.setUsername("root");
        config.setPassword("your_password");
        config.setMaximumPoolSize(10);
        dataSource = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

---

## 7. 完整 CRUD 示例

```java
package org.example.MyJDBC;

import java.sql.*;

public class StudentDAO {
    public void findAll() {
        String sql = "SELECT * FROM student";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {
            while (rs.next()) {
                System.out.printf("id=%d, name=%s, age=%d%n",
                    rs.getInt("id"), rs.getString("name"), rs.getInt("age"));
            }
        } catch (SQLException e) { e.printStackTrace(); }
    }

    public void insert(String name, int age) {
        String sql = "INSERT INTO student(name, age) VALUES(?, ?)";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, name);
            pstmt.setInt(2, age);
            pstmt.executeUpdate();
        } catch (SQLException e) { e.printStackTrace(); }
    }
}
```

---

## 8. 总结

| 知识点 | 要点 |
|--------|------|
| **JDBC 本质** | 一组数据库操作的 Java 接口，厂商提供实现（驱动） |
| **连接** | `DriverManager.getConnection(url, user, pwd)` |
| **Statement** | 拼接 SQL，有注入风险 ❌ |
| **PreparedStatement** | `?` 占位符 + 预编译，防注入 ✅ |
| **事务** | `setAutoCommit(false)` → commit / rollback |
| **连接池** | HikariCP，复用连接，性能大幅提升 |
| **try-with-resources** | 自动关闭资源 |

---

*对应包名：`org.example.MyJDBC`*
*最后更新: 2026-09-25*
