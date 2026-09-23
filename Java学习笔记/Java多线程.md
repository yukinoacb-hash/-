# 多线程

## 进程 vs 线程

```
进程 = 一个运行的程序（如 Chrome）
线程 = 进程里的一个任务（如 Chrome 里的一个标签页）
```

## 创建线程的两种方式

```java
// 方式一：继承 Thread
class MyThread extends Thread {
    public void run() { System.out.println("线程跑起来了"); }
}
new MyThread().start();

// 方式二：实现 Runnable（推荐，不影响继承其他类）
Thread t = new Thread(() -> {
    System.out.println("线程跑起来了");
});
t.start();
```

## 主线程

main 方法本身就跑在主线程上，不用自己创建。

## 线程安全

多个线程同时操作同一个数据，会丢数据。

```java
private static int count = 0;

// 问题：count++ 底层分三步，两个线程同时执行会丢值
// 解决：synchronized
private static synchronized void increment() {
    count++;
}
```

## 线程顺序

线程执行顺序由操作系统决定，每次跑结果可能不同。
