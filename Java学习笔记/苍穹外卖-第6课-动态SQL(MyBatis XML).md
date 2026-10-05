# 苍穹外卖 · 第 6 课：动态 SQL（MyBatis XML）

## 📖 知识点概述（一句话）

**动态 SQL = SQL 语句由程序"按条件拼装"**：传了值就拼进 SQL，没传就不拼。
MyBatis 用它来写"部分字段更新""多条件搜索"这类 SQL。

---

## 🧠 为什么需要动态 SQL

### 起因：注解版的"全字段更新"会毁数据

```java
@Update("UPDATE employee SET username=#{username}, name=#{name}, password=#{password}, ... WHERE id=#{id}")
```

前端只传了 `name` 和 `phone`，其他字段就是 `null`，SQL 实际变成：

```sql
UPDATE employee SET username=NULL, name='张三', password=NULL, ... WHERE id=7
```

→ `username` 是 `NOT NULL`，直接 500；就算能跑，密码也被清空了。

**根本原因**：注解版 SQL 是"死"的 —— 写死的字符串，**列必须全出现**。

---

## 🧠 关键要点

### 1. 注解 vs XML

| | 注解（`@Select/@Insert/@Update/@Delete`） | XML 映射文件 |
|---|---|---|
| 适合 | 简单、固定不变的 SQL | **动态 SQL、复杂 SQL** |
| 写动态标签 | 要包在 `<script>` 里，很丑 | 原生支持，清爽 |
| 官方态度 | 简单场景够用 | 复杂 SQL 的推荐方案 |

**两者可以共存**：同一个 Mapper 接口里，简单的用注解，复杂的用 XML。

### 2. 两个核心标签

**`<if>` —— 条件判断**

```xml
<if test="name != null and name != ''">
    name = #{name},
</if>
```
- `test` 里是 **OGNL 表达式**，写的是 Java 对象的属性名
- 条件成立 → 这段 SQL 才拼进去；不成立 → 整段消失
- 字符串要同时判 `!= null`（是不是空）和 `!= ''`（是不是空串）

**`<set>` —— 自动生成 SET 关键字**

```xml
<update id="update">
    UPDATE employee
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="phone != null">phone = #{phone},</if>
    </set>
    WHERE id = #{id}
</update>
```

`<set>` 帮你做两件事：
1. 自动加上 `SET` 关键字
2. **自动删掉最后多余的逗号** ← 这是它最大的价值！

（如果手写，最后一个条件不成立时就会变成 `SET name = ?, WHERE` 这种语法错误。）

### 3. 其他常用标签

| 标签 | 作用 |
|---|---|
| `<if>` | 条件拼接 |
| `<set>` | UPDATE 的 SET 部分，自动去尾逗号 |
| `<where>` | 查询的 WHERE 部分，自动去首 AND/OR |
| `<foreach>` | 遍历集合（比如 `id IN (1,2,3)`） |
| `<trim>` | 自定义前后缀/去除规则（`<set>` 的底层就是它） |

### 4. XML 与接口的绑定规则（重要）

```xml
<mapper namespace="com.sky.skytakeout.mapper.EmployeeMapper">   <!-- = 接口的全限定名 -->
    <update id="update">                                          <!-- = 接口里的方法名 -->
        ...
    </update>
</mapper>
```

- `namespace` = **接口的全限定类名**（包名 + 类名），一个字母都不能错
- `id` = **接口里对应的方法名**
- 两者对上，MyBatis 才知道"这个方法该执行哪段 SQL"

**绑定完成后，接口上原来的注解要删掉**，否则会冲突报错。

### 5. XML 文件放哪
项目在 `application.yml` 里已经配置好了：

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
```

意思是：**去 classpath 的 `mapper` 目录下找所有 `.xml`**。

👉 所以文件必须放在 `src/main/resources/mapper/` 目录下，文件名随意（习惯上叫 `EmployeeMapper.xml`）。

---

## ⚠️ 注意事项 / 常见坑

### 坑 1：`WHERE` 不能动态化，否则可能全表更新 💀

```xml
<set>...</set>
WHERE id = #{id}          <!-- ✓ 必须写死，id 是更新的定位条件 -->
```

**永远不要让主键条件可以被"跳过"**。一旦 `<if>` 让 WHERE 失效，就是全表更新。

### 坑 2：命名空间/方法名写错 → `Invalid bound statement (not found)`
这是 XML 版最典型的报错，95% 是这两个地方打错了。

### 坑 3：XML 里的 `<` 要转义
XML 中 `<` 是标签开始符，写 `age < 10` 要写成 `age &lt; 10`（或用 `<![CDATA[ ]]>` 包起来）。

### 坑 4：改完 XML 也要重新编译
XML 在 `resources` 里，**同样会被复制到 `target/classes`**。
所以改完 XML 也要 ⬛ 停止 → ▶ 重新运行，否则跑的还是旧的。

---

## 🔗 相关链接

- 上一课：[苍穹外卖-第5课-修改员工PUT](苍穹外卖-第5课-修改员工PUT.md)
