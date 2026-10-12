# 反射

## 获取 Class 对象

```java
Class<?> clazz = Class.forName("org.example.User");
```

## 创建对象

```java
// 无参构造
User u = (User) clazz.getDeclaredConstructor().newInstance();

// 有参构造
Constructor<?> c = clazz.getDeclaredConstructor(String.class, int.class);
User u = (User) c.newInstance("张三", 20);
```

## 调用方法

```java
// 单个方法（已知方法名）
Method m = clazz.getMethod("sayHello");
m.invoke(u);

// 私有方法需加 setAccessible(true)
Method m2 = clazz.getDeclaredMethod("getInfo");
m2.setAccessible(true);
String result = (String) m2.invoke(u);

// 查看所有方法（忘了方法名时）
for (Method m : clazz.getDeclaredMethods()) {
    System.out.println(m.getName());
}
```

## 操作字段

```java
Field f = clazz.getDeclaredField("name");
f.setAccessible(true);
f.set(u, "李四");          // 赋值
System.out.println(f.get(u)); // 取值
```

## 必要性

```
new            → 编译时写死
Class.forName  → 运行时动态（从配置文件等字符串加载）
```

反射把"硬编码"变成"可配置"，框架底层都是这么干的。
