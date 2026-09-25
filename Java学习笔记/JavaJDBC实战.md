# Java JDBC 实战 — 学生成绩管理系统

> 动手做一个完整的小项目，把 JDBC 的 CRUD、事务、连接池全部练一遍。

---

## 📋 项目结构

```
org.example.MyJDBC
├── DBUtil.java          — 连接池工具类（HikariCP）
├── Student.java         — 实体类
├── StudentDAO.java      — 数据访问层（CRUD）
├── StudentService.java  — 业务层（事务管理）
└── Main.java            — 控制台菜单（运行入口）
```

---

## 1. 准备表结构

在 Navicat 中执行（如果 student 表已有，加 score 字段）：

```sql
-- 如果已有 student 表，只加成绩字段
ALTER TABLE student ADD COLUMN score DECIMAL(5,2) DEFAULT 0.00 COMMENT "成绩";

-- 如果是新建表
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    age INT DEFAULT 0,
    score DECIMAL(5,2) DEFAULT 0.00
);

-- 插入测试数据
INSERT INTO student(name, age, score) VALUES("张三", 20, 85.5);
INSERT INTO student(name, age, score) VALUES("李四", 21, 92.0);
INSERT INTO student(name, age, score) VALUES("王五", 19, 76.5);
INSERT INTO student(name, age, score) VALUES("赵六", 22, 88.0);
```

---

## 2. DBUtil — 连接池工具类

> 封装 HikariCP，整个项目共用这一个类获取连接。

```java
package org.example.MyJDBC;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

/**
 * 数据库连接工具类（HikariCP 连接池）
 * 
 * 使用方式：
 *   Connection conn = DBUtil.getConnection();
 *   // 用完 conn.close() 是归还到连接池，不是真的关闭
 */
public class DBUtil {
    private static final DataSource DATA_SOURCE;

    static {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/java_test"
                + "?useSSL=false&serverTimezone=Asia/Shanghai"
                + "&characterEncoding=utf8");
        config.setUsername("root");
        config.setPassword("your_password");   // ← 改成你的密码
        config.setMaximumPoolSize(10);          // 最大连接数
        config.setMinimumIdle(5);               // 最小空闲数
        config.setConnectionTimeout(30000);     // 等连接的超时时间(ms)
        config.setIdleTimeout(600000);          // 空闲多久被回收(ms)
        DATA_SOURCE = new HikariDataSource(config);
    }

    /** 从连接池获取一个连接 */
    public static Connection getConnection() throws SQLException {
        return DATA_SOURCE.getConnection();
    }
}
```

---

## 3. Student — 实体类

> 一张表对应一个 Java 类，字段一一映射。

```java
package org.example.MyJDBC;

/**
 * 学生实体类 — 跟 student 表字段一一对应
 */
public class Student {
    private int id;
    private String name;
    private int age;
    private double score;

    // 构造方法
    public Student() {}

    public Student(String name, int age, double score) {
        this.name = name;
        this.age = age;
        this.score = score;
    }

    // Getter / Setter
    public int getId() { return id; }
    public void setId(int id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }

    public double getScore() { return score; }
    public void setScore(double score) { this.score = score; }

    @Override
    public String toString() {
        return String.format("| %-3d | %-6s | %-3d | %-6.1f |",
                id, name, age, score);
    }
}
```

---

## 4. StudentDAO — 数据访问层

> 只负责 SQL 操作，不处理业务逻辑。

```java
package org.example.MyJDBC;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;

/**
 * 数据访问层 — 只写 SQL，不写业务逻辑
 *
 * 核心原则：
 * 1. 所有 SQL 都用 PreparedStatement（防注入）
 * 2. 所有资源都用 try-with-resources（自动关闭）
 * 3. 连接从连接池借，用完归还
 */
public class StudentDAO {

    // ==================== 增 ====================

    /** 插入学生（返回自增 ID） */
    public int insert(Student student) throws SQLException {
        String sql = "INSERT INTO student(name, age, score) VALUES(?, ?, ?)";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql,
                     Statement.RETURN_GENERATED_KEYS)) {

            pstmt.setString(1, student.getName());
            pstmt.setInt(2, student.getAge());
            pstmt.setDouble(3, student.getScore());

            pstmt.executeUpdate();

            // 获取自增 ID
            ResultSet rs = pstmt.getGeneratedKeys();
            if (rs.next()) {
                int id = rs.getInt(1);
                student.setId(id);
                return id;
            }
            return 0;
        }
    }

    // ==================== 删 ====================

    /** 按 ID 删除 */
    public int deleteById(int id) throws SQLException {
        String sql = "DELETE FROM student WHERE id = ?";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setInt(1, id);
            return pstmt.executeUpdate();  // 返回影响行数
        }
    }

    // ==================== 改 ====================

    /** 更新学生信息 */
    public int update(Student student) throws SQLException {
        String sql = "UPDATE student SET name = ?, age = ?, score = ? WHERE id = ?";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setString(1, student.getName());
            pstmt.setInt(2, student.getAge());
            pstmt.setDouble(3, student.getScore());
            pstmt.setInt(4, student.getId());

            return pstmt.executeUpdate();
        }
    }

    // ==================== 查 ====================

    /** 查询所有学生 */
    public List<Student> findAll() throws SQLException {
        String sql = "SELECT * FROM student ORDER BY id";
        List<Student> list = new ArrayList<>();

        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {

            while (rs.next()) {
                list.add(mapRow(rs));
            }
        }
        return list;
    }

    /** 按 ID 查询 */
    public Student findById(int id) throws SQLException {
        String sql = "SELECT * FROM student WHERE id = ?";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setInt(1, id);
            ResultSet rs = pstmt.executeQuery();
            if (rs.next()) {
                return mapRow(rs);
            }
            return null;
        }
    }

    /** 按姓名模糊查询 */
    public List<Student> findByName(String keyword) throws SQLException {
        String sql = "SELECT * FROM student WHERE name LIKE ?";
        List<Student> list = new ArrayList<>();

        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setString(1, "%" + keyword + "%");  // 模糊查询
            ResultSet rs = pstmt.executeQuery();
            while (rs.next()) {
                list.add(mapRow(rs));
            }
        }
        return list;
    }

    // ==================== 工具 ====================

    /**
     * 将 ResultSet 当前行映射为 Student 对象
     * 提取成方法，避免每个查询都重复写 getInt/getString
     */
    private Student mapRow(ResultSet rs) throws SQLException {
        Student s = new Student();
        s.setId(rs.getInt("id"));
        s.setName(rs.getString("name"));
        s.setAge(rs.getInt("age"));
        s.setScore(rs.getDouble("score"));
        return s;
    }
}
```

---

## 5. StudentService — 业务层（事务）

> 处理业务逻辑，**需要事务保证的操作放这里**。

```java
package org.example.MyJDBC;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

/**
 * 业务层 — 处理业务逻辑，管理事务
 *
 * 核心原则：
 * 1. 一个业务方法内多条 SQL → 手动管理事务
 * 2. 纯查询不需要事务
 * 3. 事务操作：获取连接 → setAutoCommit(false)
 *    → 所有 SQL 用同一个 conn → commit → finally 归还
 */
public class StudentService {
    private final StudentDAO studentDAO = new StudentDAO();

    // ==================== 无事务操作 ====================

    public List<Student> findAll() throws SQLException {
        return studentDAO.findAll();
    }

    public Student findById(int id) throws SQLException {
        return studentDAO.findById(id);
    }

    public List<Student> findByName(String name) throws SQLException {
        return studentDAO.findByName(name);
    }

    // ==================== 有事务操作 ====================

    /**
     * 新增学生（单条 SQL，自动事务就够了）
     */
    public void addStudent(Student student) throws SQLException {
        studentDAO.insert(student);
        System.out.println("添加成功！ID = " + student.getId());
    }

    /**
     * 修改学生信息
     */
    public void updateStudent(Student student) throws SQLException {
        int rows = studentDAO.update(student);
        System.out.println(rows > 0 ? "修改成功！" : "未找到该学生");
    }

    /**
     * 删除学生
     */
    public void deleteStudent(int id) throws SQLException {
        int rows = studentDAO.deleteById(id);
        System.out.println(rows > 0 ? "删除成功！" : "未找到该学生");
    }

    /**
     * ⭐ 事务实战：批量插入学生
     *
     * 需求：要么全部插入成功，要么一个都不插
     * 场景：Excel 导入学生名单，中间有一条数据有问题就全部回滚
     */
    public void batchInsert(List<Student> students) throws SQLException {
        String sql = "INSERT INTO student(name, age, score) VALUES(?, ?, ?)";

        // 事务必须手动获取连接，让多条 SQL 用同一个连接
        Connection conn = null;
        try {
            conn = DBUtil.getConnection();
            conn.setAutoCommit(false);  // ⚠️ 关闭自动提交

            try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
                for (Student s : students) {
                    pstmt.setString(1, s.getName());
                    pstmt.setInt(2, s.getAge());
                    pstmt.setDouble(3, s.getScore());
                    pstmt.executeUpdate();
                }
            }

            conn.commit();  // ✅ 全部成功 → 提交
            System.out.println("批量插入成功！共 " + students.size() + " 条");
        } catch (Exception e) {
            if (conn != null) {
                conn.rollback();  // ❌ 有异常 → 全部回滚
            }
            System.out.println("批量插入失败，已全部回滚！");
            throw e;
        } finally {
            if (conn != null) {
                conn.setAutoCommit(true);  // 恢复自动提交（归还连接前重置）
                conn.close();              // 归还到连接池
            }
        }
    }

    /**
     * ⭐ 事务实战：调整成绩（从一个学生扣分，给另一个学生加分）
     *
     * 需求：A 扣 5 分给 B 加 5 分（总分不变）
     * 事务保证：扣和加要么都成功，要么都回滚
     */
    public void transferScore(int fromId, int toId, double score)
            throws SQLException {
        String sql1 = "UPDATE student SET score = score - ? WHERE id = ?";
        String sql2 = "UPDATE student SET score = score + ? WHERE id = ?";

        Connection conn = null;
        try {
            conn = DBUtil.getConnection();
            conn.setAutoCommit(false);

            // 扣分
            try (PreparedStatement pstmt = conn.prepareStatement(sql1)) {
                pstmt.setDouble(1, score);
                pstmt.setInt(2, fromId);
                int rows = pstmt.executeUpdate();
                if (rows == 0) {
                    throw new SQLException("扣分学生不存在！");
                }
            }

            // 加分
            try (PreparedStatement pstmt = conn.prepareStatement(sql2)) {
                pstmt.setDouble(1, score);
                pstmt.setInt(2, toId);
                int rows = pstmt.executeUpdate();
                if (rows == 0) {
                    throw new SQLException("加分学生不存在！");
                }
            }

            conn.commit();
            System.out.println("成绩调整成功！" + score + " 分已转移");
        } catch (Exception e) {
            if (conn != null) {
                conn.rollback();
            }
            System.out.println("成绩调整失败，已回滚！原因：" + e.getMessage());
            throw e;
        } finally {
            if (conn != null) {
                conn.setAutoCommit(true);
                conn.close();
            }
        }
    }
}
```

---

## 6. Main — 控制台菜单

> 给用户一个交互菜单，调用 Service 层完成操作。

```java
package org.example.MyJDBC;

import java.util.Arrays;
import java.util.List;
import java.util.Scanner;

/**
 * 学生成绩管理系统 — 控制台入口
 *
 * 运行后输入数字选择功能：
 *   1 查看所有学生
 *   2 按 ID 查询
 *   3 按姓名搜索
 *   4 添加学生
 *   5 修改学生
 *   6 删除学生
 *   7 调整成绩（事务）
 *   8 批量插入（事务）
 *   0 退出
 */
public class Main {
    private static final StudentService service = new StudentService();
    private static final Scanner scanner = new Scanner(System.in);

    public static void main(String[] args) {
        System.out.println("═══════════════════════════════");
        System.out.println("  学生成绩管理系统 (JDBC 版)");
        System.out.println("═══════════════════════════════");

        while (true) {
            printMenu();
            System.out.print("请选择操作：");
            try {
                int choice = Integer.parseInt(scanner.nextLine());
                switch (choice) {
                    case 1 -> listAll();
                    case 2 -> findById();
                    case 3 -> findByName();
                    case 4 -> addStudent();
                    case 5 -> updateStudent();
                    case 6 -> deleteStudent();
                    case 7 -> transferScore();
                    case 8 -> batchInsert();
                    case 0 -> {
                        System.out.println("感谢使用，再见！");
                        return;
                    }
                    default -> System.out.println("无效选项，请重新输入");
                }
            } catch (NumberFormatException e) {
                System.out.println("请输入数字！");
            } catch (Exception e) {
                System.out.println("操作失败：" + e.getMessage());
                e.printStackTrace();
            }
            System.out.println();
        }
    }

    private static void printMenu() {
        System.out.println("───────────────────────────────");
        System.out.println(" 1. 查看所有学生");
        System.out.println(" 2. 按 ID 查询");
        System.out.println(" 3. 按姓名搜索");
        System.out.println(" 4. 添加学生");
        System.out.println(" 5. 修改学生信息");
        System.out.println(" 6. 删除学生");
        System.out.println(" 7. ⭐ 调整成绩（事务）");
        System.out.println(" 8. ⭐ 批量插入（事务）");
        System.out.println(" 0. 退出");
        System.out.println("───────────────────────────────");
    }

    private static void listAll() throws Exception {
        List<Student> list = service.findAll();
        if (list.isEmpty()) {
            System.out.println("暂无学生数据");
            return;
        }
        printTable(list);
    }

    private static void findById() throws Exception {
        System.out.print("请输入学生 ID：");
        int id = Integer.parseInt(scanner.nextLine());
        Student s = service.findById(id);
        if (s == null) {
            System.out.println("未找到该学生");
        } else {
            printTable(List.of(s));
        }
    }

    private static void findByName() throws Exception {
        System.out.print("请输入姓名关键字：");
        String name = scanner.nextLine();
        List<Student> list = service.findByName(name);
        if (list.isEmpty()) {
            System.out.println("未找到匹配的学生");
        } else {
            printTable(list);
        }
    }

    private static void addStudent() throws Exception {
        System.out.print("姓名：");
        String name = scanner.nextLine();
        System.out.print("年龄：");
        int age = Integer.parseInt(scanner.nextLine());
        System.out.print("成绩：");
        double score = Double.parseDouble(scanner.nextLine());

        service.addStudent(new Student(name, age, score));
    }

    private static void updateStudent() throws Exception {
        System.out.print("要修改的学生 ID：");
        int id = Integer.parseInt(scanner.nextLine());

        Student s = service.findById(id);
        if (s == null) {
            System.out.println("未找到该学生");
            return;
        }

        System.out.println("当前信息：" + s);
        System.out.print("新姓名（回车不修改）：");
        String name = scanner.nextLine();
        if (!name.isEmpty()) s.setName(name);

        System.out.print("新年龄（回车不修改）：");
        String age = scanner.nextLine();
        if (!age.isEmpty()) s.setAge(Integer.parseInt(age));

        System.out.print("新成绩（回车不修改）：");
        String score = scanner.nextLine();
        if (!score.isEmpty()) s.setScore(Double.parseDouble(score));

        service.updateStudent(s);
    }

    private static void deleteStudent() throws Exception {
        System.out.print("要删除的学生 ID：");
        int id = Integer.parseInt(scanner.nextLine());
        service.deleteStudent(id);
    }

    private static void transferScore() throws Exception {
        System.out.println("=== 调整成绩（事务演示）===");
        System.out.print("扣分学生 ID：");
        int fromId = Integer.parseInt(scanner.nextLine());
        System.out.print("加分学生 ID：");
        int toId = Integer.parseInt(scanner.nextLine());
        System.out.print("转移分数：");
        double score = Double.parseDouble(scanner.nextLine());

        service.transferScore(fromId, toId, score);
    }

    private static void batchInsert() throws Exception {
        System.out.println("=== 批量插入（事务演示）===");
        System.out.println("输入格式：姓名,年龄,成绩（一行一个，空行结束）");
        System.out.println("示例：小明,18,90.5");

        List<Student> students = new java.util.ArrayList<>();
        String line;
        while (!(line = scanner.nextLine()).isEmpty()) {
            String[] parts = line.split(",");
            if (parts.length == 3) {
                String name = parts[0].trim();
                int age = Integer.parseInt(parts[1].trim());
                double score = Double.parseDouble(parts[2].trim());
                students.add(new Student(name, age, score));
            } else {
                System.out.println("格式错误，请按 姓名,年龄,成绩 输入");
            }
        }

        if (!students.isEmpty()) {
            service.batchInsert(students);
        }
    }

    /** 打印表格 */
    private static void printTable(List<Student> list) {
        System.out.println("+------+--------+------+--------+");
        System.out.println("| ID   | 姓名   | 年龄 | 成绩   |");
        System.out.println("+------+--------+------+--------+");
        for (Student s : list) {
            System.out.println(s);
        }
        System.out.println("+------+--------+------+--------+");
    }
}
```

---

## 7. ⭐ 核心要点总结

### 7.1 三层架构

```
Main (界面层) → StudentService (业务层) → StudentDAO (数据层) → DB
                ↑ 管事务              ↑ 管 SQL
```

### 7.2 什么时候用事务？

| 场景 | 是否要事务 | 原因 |
|------|-----------|------|
| 查询一个学生 | ❌ 不需要 | 不修改数据 |
| 添加一个学生 | ❌ 不需要 | 单条 SQL，自动事务就够了 |
| **批量插入** | **✅ 必须** | 中间出错要全部回滚 |
| **调整成绩（扣分+加分）** | **✅ 必须** | 两条 SQL 要么都成功要么都回滚 |

### 7.3 事务模板（背下来）

```java
Connection conn = null;
try {
    conn = DBUtil.getConnection();
    conn.setAutoCommit(false);       // 1. 关自动提交

    // ... 多条 SQL 操作 ...

    conn.commit();                    // 2. 全部成功 → 提交
} catch (Exception e) {
    if (conn != null) conn.rollback(); // 3. 出错 → 回滚
    throw e;
} finally {
    if (conn != null) {
        conn.setAutoCommit(true);     // 4. 恢复自动提交
        conn.close();                 // 5. 归还连接
    }
}
```

### 7.4 常用 PreparedStatement 方法

```java
pstmt.setString(1, value);
pstmt.setInt(2, value);
pstmt.setDouble(3, value);
pstmt.setObject(4, value);   // 通用，自动推断类型
```

### 7.5 常用 ResultSet 方法

```java
rs.next()                    // 下移一行，没数据返回 false
rs.getInt("列名")
rs.getString("列名")
rs.getDouble("列名")
rs.getObject("列名")         // 通用
```

---

## 8. 运行方法

### 在 IDEA 中运行

1. **包名**：`org.example.MyJDBC`
2. **Maven 依赖**（pom.xml）：

```xml
<dependencies>
    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.4.0</version>
    </dependency>

    <!-- HikariCP 连接池 -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.1.0</version>
    </dependency>
</dependencies>
```

3. **修改密码**：`DBUtil.java` 中 `your_password` 改成你的 MySQL 密码
4. **运行** `Main.java`，选择菜单操作

---

## 9. 拓展练习（自测）

做完上面的基础版，试试这些拓展：

1. **分页查询**：`LIMIT ?, ?` 实现分页，参数传 offset 和 pageSize
2. **按成绩排序**：`ORDER BY score DESC` 显示排名
3. **统计功能**：查询平均分、最高分、最低分（`AVG` / `MAX` / `MIN`）
4. **防止 XSS**：尝试输入 `<script>alert("xss")</script>`，看 PreparedStatement 会不会拦截
5. **加一个 Course 表**：学生选课，练多表联查

---

## 10. JDBC 学习红线

| 级别 | 要求 |
|------|------|
| **必须掌握 ✅** | PreparedStatement CRUD、事务模板、try-with-resources |
| **必须理解 🔵** | SQL 注入原理、连接池作用、三层架构分层 |
| **了解即可 📖** | Statement（知道有注入风险）、CallableStatement、Blob/Clob |
| **不用深究 ❌** | 自己手写连接池、DatabaseMetaData、RowSet |

> **JDBC 学到「能不看书写出完整 CRUD + 事务 + 连接池」就够用了**。  
> 后续学 MyBatis 时框架会帮你封装掉重复代码。

---

*对应包名：`org.example.MyJDBC`*
*最后更新: 2026-09-25*
