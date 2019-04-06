---
layout: post
title: CountDownLatch使用
toc: true
cover: /img/cover/why-is-binary.png
tags: ['java','并发']
category: Java
---

# CountDownLatch使用
> `CountDownLatch`是java中的一个同步工具类.用于对线程的阻塞和唤醒

## 使用

### 1.初始化

在初始化创建对象时需要传入一个`int`类型的`count`值. 

### 2.方法

+ **`getCount`** 获取当前对象的`count` 值.
+ **`countDown`** 将当前对象的`count` 值减1.
+ **`await`**调用`CountDownLatch`对象的`await`方法时会阻塞当前线程,当这个对象的`count`为0时会唤醒这个对象调用`await`方法阻塞的所有线程 .
+ **`await(long time,TimeUnit unit)`** `await` 方法重载,在如果阻塞的时间超过定义的超时时间,线程自动唤醒

**代码示例:**

```java
package concurrent;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
     public class Test {
        private CountDownLatch countDownLatch = new CountDownLatch(1);
        public void method1(String name) {
            try {
                //countDownLatch.await();  //阻塞线程,到countDownLatch对象的count等于0时唤醒
                 countDownLatch.await(10L,TimeUnit.SECONDS); //阻塞线程,如果在10秒后还没有被唤醒,将自动唤醒
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(System.currentTimeMillis() + " " + Thread.currentThread().getName() + " - " + name + " 被唤醒");
        }
    
        public void method2(String name) {
            System.out.println(Thread.currentThread().getName() + "  " + name);
            try {
                TimeUnit.SECONDS.sleep(5);
                countDownLatch.countDown(); // 将count值减1
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    
        public static void main(String[] args) throws InterruptedException {
            Test test = new Test();
            new Thread(() -> {
                test.method1("method1");
            }).start();
            new Thread(() -> {
                test.method2("method2");
            }).start();
            new Thread(() -> {
                test.method1("method3");
            }).start();
        }
    
}
```





### 使用场景

在需要线程同步的场景下使用,比使用`wait` `notify` 方法更为灵活 
​      


