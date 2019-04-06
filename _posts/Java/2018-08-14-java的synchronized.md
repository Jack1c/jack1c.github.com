---
layout: post
title: java的synchronized
toc: true
cover: /img/cover/67a6be23d10660deb64be6861dbf8934.jpg
tags: ['java','并发']
category: Java
---


# java的synchronized

synchronized作用于代码块, 在java中用于保证代码的同步.  在java5之前

### 怎么用?

使用 synchronized 时需要一个对象作为锁. 

**作用范围:** 

+ 代码块, 手动指定一个对象作为锁
+ 方法 
  + 实例方法, 调用该方法的当前对象作为锁
  + 静态方法  该方法所在的类的class对象作为锁



使用了相同锁对象的synchronized的代码,在任何时刻只有一个线程执行,当一个线程执行代码块时会先获取对象锁,如果获取不到时线程进入阻塞状态, 直到其他的线程释放锁对象,再去获取,知道获取到锁对象才能执行,否则一直阻塞下去.



**代码实例:**

```java
import java.time.LocalTime;

public class TestSychronized {
    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (TestSychronized.class) {
                System.out.println(Thread.currentThread().getName() + " " + LocalTime.now());
                try {
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "t1").start();

        new Thread(() -> {
            synchronized (TestSychronized.class) {
                System.out.println(Thread.currentThread().getName() + " " + LocalTime.now());
            }
        }, "t2").start();
    }
}
```

**执行结果:**

```
t1 12:08:32.493
t2 12:08:42.497
```

线程t1先获取锁进入执行,在执行完打印语句后进入睡眠状态,此时t2线程获取不到锁进入阻塞状态.线程t1睡眠结束synchronized代码的执行释放锁, 线程t2获取到锁,执行打印语句. 



**释放锁的条件:**

+ 代码执行完自动释放
+ 抛出异常时自动释放
+ 调用锁对象的`wait`方式时∂

### 原理是什么?

#### 字节码

synchronized代码块在编译成字节码后,有两个指令对应.

**代码实例:**

```java
import java.time.LocalTime;

public class TestSychronized2 {
    private static Object lock = new Object();
    public static void main(String[] args) {
        synchronized (lock){

        }
    }

    public static synchronized void test(){}
}
```

在编译后使用 ` javap -c -verbose TestSychronized2 ` 查看编译后的字节码信息

```
Classfile /code/java/toyed-demo/code-test/src/main/java/TestSychronized2.class
  Last modified Aug 18, 2018; size 565 bytes
  MD5 checksum 69b26873733e9f1c1123efaecca14b0d
  Compiled from "TestSychronized2.java"
public class TestSychronized2
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#21         // java/lang/Object."<init>":()V
   #2 = Fieldref           #4.#22         // TestSychronized2.lock:Ljava/lang/Object;
   #3 = Class              #23            // java/lang/Object
   #4 = Class              #24            // TestSychronized2
   #5 = Utf8               lock
   #6 = Utf8               Ljava/lang/Object;
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               StackMapTable
  #14 = Class              #25            // "[Ljava/lang/String;"
  #15 = Class              #23            // java/lang/Object
  #16 = Class              #26            // java/lang/Throwable
  #17 = Utf8               test
  #18 = Utf8               <clinit>
  #19 = Utf8               SourceFile
  #20 = Utf8               TestSychronized2.java
  #21 = NameAndType        #7:#8          // "<init>":()V
  #22 = NameAndType        #5:#6          // lock:Ljava/lang/Object;
  #23 = Utf8               java/lang/Object
  #24 = Utf8               TestSychronized2
  #25 = Utf8               [Ljava/lang/String;
  #26 = Utf8               java/lang/Throwable
{
  public TestSychronized2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // Field lock:Ljava/lang/Object;
         3: dup
         4: astore_1
         5: monitorenter
         6: aload_1
         7: monitorexit
         8: goto          16
        11: astore_2
        12: aload_1
        13: monitorexit
        14: aload_2
        15: athrow
        16: return
      Exception table:
         from    to  target type
             6     8    11   any
            11    14    11   any
      LineNumberTable:
        line 6: 0
        line 8: 6
        line 9: 16
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 11
          locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  public static synchronized void test();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 11: 0

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: new           #3                  // class java/lang/Object
         3: dup
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: putstatic     #2                  // Field lock:Ljava/lang/Object;
        10: return
      LineNumberTable:
        line 4: 0
}
SourceFile: "TestSychronized2.java"

```



从字节码中可以看出 当进入 synchronized 代码块时会插入一个 `monitorenter`的指令, 出代码的时候插入`monitorexit` 指令. 在方法上使用synchronized时 会给方法加一个 `ACC_SYNCHRONIZED`的标志.



#### java的对象头

+ 数组类型对象 : 3 个字存储对象头
+ 非数组对象 2个字存储对象头

| 长度     | 内容                   | 说明                             |
| -------- | ---------------------- | -------------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息等。   |
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针         |
| 32/64bit | Array length           | 数组的长度（如果当前对象是数组） |



**java对象头的Mark word 存储:**

+ 对象的HashCode 
+ 分代年龄
+ 锁标记

32为虚拟机Mark Work默认存储:

|          | 25bit          | 4bit           | 1bit 是否为偏向锁 | 2bit 锁标志位 |
| -------- | -------------- | -------------- | ----------------- | ------------- |
| 无锁状态 | 对象的hashcode | 对象的分代年龄 | 0                 | 01            |

运行期间 Mark Word 里存储的数据根据所标志位的变化而变化.

**Mark Word 可能变化的四种数据:**

|  锁状态  | 23bit  | 2bit  |     4bit     | 1bit( 是否是偏向锁) | 2bit(锁标志位) |
| :------: | ------ | :---: | :----------: | :-----------------: | -------------- |
|  偏向锁  | 线程id | Epoch | 对象分代年龄 |          1          | 01             |

|  锁状态  |          30bit           | 2bit(锁标志位) |
| :------: | :----------------------: | :------------: |
| 轻量级锁 |   指向栈中锁记录的指针   |       00       |
| 重量级锁 | 指向互斥量(重量锁)的指针 |       10       |
|  GC标记  |            空            |       11       |



**64位虚拟机,Mark Word为64bit大小,结构:**

| 锁状态 | 25bit                       | 31bit | 1bit | 4bit | 1bit(偏向锁) | 2bit(锁标志位) |
| ------ | --------------------------- | ----- | ---- | ---- | ------------ | -------------- |
| 无锁   |                             |       |      |      | 0            | 01             |
| 偏向锁 | ThreadID(54bit) Epoch(2bit) |       |      |      | 1            | 01             |



### 锁升级

在java 6 之前 monitor是依靠**操作系统内部的互斥锁**实现的, 由于需要进行用户态到内核态的切换, 所有同步操作是一个重量级的操作.在后面的jdk中 java对同步机制做了大量的调整, 通过三种不同的锁来实现: 偏斜锁, 轻量锁, 重量锁,在现代的jdk中,JVM根据不同的竞争条件对synchronized做出优化,适当做出锁的切换,这种切换称为 **锁的升级,降级** .

**1.偏向锁**

 Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

 

**偏向锁的撤销**：偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。下图中的线程1演示了偏向锁初始化的流程，线程2演示了偏向锁撤销的流程。

![偏向锁的撤销](https://raw.githubusercontent.com/Jack1c/Jack1c.github.io/master/assets/img/2018/08/%E5%81%8F%E5%90%91%E9%94%81%E7%9A%84%E6%92%A4%E9%94%80.png)

**2.轻量锁**

**轻量级锁加锁**：线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

**轻量级锁解锁**：轻量级解锁时，会使用原子的CAS操作来将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。下图是两个线程同时争夺锁，导致锁膨胀的流程图。
![轻量级锁](https://raw.githubusercontent.com/Jack1c/Jack1c.github.io/master/assets/img/2018/08/%E9%94%81%E8%86%A8%E8%83%80.png)

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

  **锁的优缺点对比**

| 锁       | 优点                                                         | 缺点                                             | 适用场景                             |
| -------- | ------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------ |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距。 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗。 | 适用于只有一个线程访问同步块场景。   |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度。                   | 如果始终得不到锁竞争的线程使用自旋会消耗CPU。    | 追求响应时间。同步块执行速度非常快。 |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU。                            | 线程阻塞，响应时间缓慢。                         | 追求吞吐量。同步块执行速度较长。     |



参考: 

[聊聊并发（二）Java SE1.6中的Synchronized]:http://ifeve.com/java-synchronized/#header

