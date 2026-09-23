# 反射

## 什么是反射

在运行时动态地获取类的信息、操作对象的属性和方法，而不是在编译时就写死。

## 获取 Class 对象的三种方式

```java
Class<User> clazz1 = User.class;              // 方式一
Class<?> clazz2 = new User().getClass();       // 方式二
Class<?> clazz3 = Class.forName("org.example.User"); // 方式三（最灵活）
```

## 常用反射操作

```java
// 创建对象
Object obj = clazz.getDeclaredConstructor().newInstance();

// 调用方法
Method method = clazz.getMethod("sayHello");
method.invoke(obj);

// 操作字段（包括私有）
Field field = clazz.getDeclaredField("name");
field.setAccessible(true);
field.set(obj, "张三");
```

## 为什么需要反射

`new` 是编译时写死，反射是运行时动态。框架在编写时不知道你要操作什么类，只能在运行时靠反射去加载。

## 注解

```java
@Retention(RetentionPolicy.RUNTIME)   // 运行时可用
@Target(ElementType.METHOD)           // 用在方法上
public @interface Log {
    String value() default "";
}
```

## 注解的作用：标记 + 框架自动处理

```java
@Log("添加用户")     // 只贴标签
public void addUser() { /* 业务代码 */ }
```

框架用反射读取注解，自动在方法前后插入日志，不需要手动写重复代码。
