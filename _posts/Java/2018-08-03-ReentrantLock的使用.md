---
layout: post
title: ReentrantLock的使用
toc: true
cover: /img/cover/3abe2812cb7aeb01509446bdad61f4cd.jpg
tags: ['java','并发','锁']
category: Java
---

## 1.ReentrantLock的使用

ReentrantLock提供和synchronized相同的功能,但是比它更加灵活.

### 1.1用法

使用 `lock()` 方法加锁,`unlock`方法解锁,锁定的代码写在这两个方法之间

``` java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
public class TestLock {
    private ReentrantLock lock = new ReentrantLock();
    void m1() {
        //加锁
        lock.lock();
        for (int i = 0; i < 1; i++) {
            try {
                TimeUnit.SECONDS.sleep(5);
                System.out.println("m1 : " + i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println(" m1 ");
        lock.unlock();
    }
    void m2() {
        lock.lock();
        System.out.println("m2  ");
        lock.unlock();
    }
    public static void main(String[] args) {
        TestLock testLock = new TestLock();
        new Thread(testLock::m1).start();
        new Thread(testLock::m2).start();
        System.out.println("main Thread");
    }
}
```

### 1.2 tryLock

使用`tryLock()`方法可以尝试获取锁,根据获取的结果执行不同代码.

`tryLock`方法还可以指定超时时间, 在超时时间之内没有获取锁时阻塞,直到获取到锁或者超时才往后执行. 

```java
void m2() {
        //boolean locked = lock.tryLock(); //尝试获取锁
  		 locked = lock.tryLock(4, TimeUnit.SECONDS); //在4秒内进行尝试锁定
        if (!locked){
            System.out.println("没有获取到锁");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            m2();
        }else {
            System.out.println("m2  ");
          	lock.unlock();
        }
    }
```

### 1.3 lockInterruptibly方法

调用`lockInterreuptibly`方法而没有获取到锁阻塞时, 可以别的线程打断 . 

```java
 void m2() {
        try {
            //和lock方法功能相同,在阻塞时可由其他线程打断
            lock.lockInterruptibly();
            System.out.println(" m2 ");
            lock.unlock();
        } catch (InterruptedException e) {
            System.out.println("m2 被打断");
            e.printStackTrace();
        }
    }
    public static void main(String[] args) throws Exception {
        TestLock3 testLock = new TestLock3();
        new Thread(testLock::m1).start();
        Thread thread2 = new Thread(testLock::m2);
        thread2.start();
        System.out.println("main Thread");
        for (int i = 0; i < 10; i++) {
            if (i == 10) {
                //如果thread2阻塞时 可以打断thread2
                thread2.interrupt();
            }
            TimeUnit.SECONDS.sleep(1);
        }
    }
```

### 1.4 condition

ReentrantLock的`newCondition`方法能够创建一个Condition对象.通过condition对象可以唤醒指定的线程. 

使用`condition`的`await()`方法等待,在调用它的`signalAll`方法可以唤醒之前等待的线程.

```java
package org.juc;

import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
public class Condition2<E> {
    final private LinkedList<E> elementList = new LinkedList<>();
    final private int Max = 10; //指定元素个数
    private int size = 0;  //容器元素个数
    private Lock lock = new ReentrantLock();
    private Condition providerCondition = lock.newCondition(); // 生产者线程condition
    private Condition consumerCondition = lock.newCondition(); // 消费者线程condition

    public void put(E e) {
        try {
            lock.lock();
            while (size == Max) {
                try {
                    providerCondition.await(); //让所有的生产者线程(当前线程)等待
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
            }
            elementList.add(e);
            size++;
            consumerCondition.signalAll(); //叫醒所有的消费线程
        } finally {
            lock.unlock();
        }
    }
    public E get() {
        E e = null;
        try {
            lock.lock();
            while (size == 0) {
                try {
                    consumerCondition.await();  //让所有的消费者线程(当前线程)等待
                } catch (InterruptedException ecption) {
                    ecption.printStackTrace();
                }
            }
            e = elementList.removeFirst();
            size--;
            providerCondition.signalAll();  //唤醒所有等待生产者线程
        } finally {
            lock.unlock();
            return e;
        }
    }
    public static void main(String[] args) {
        Condition2<String> stringCondition = new Condition2<>();
        /**
         * 消费者线程
         */
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 5; j++) {
                    System.out.println("当前线程[" + Thread.currentThread().getName() + "] :" + stringCondition.get());
                }
            }, "c-" + i).start();
        }
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        /**
         * 生产者线程
         */
        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                for (int j = 0; j < 25; j++) {
                    stringCondition.put(Thread.currentThread().getName() + "-" + j);
                }
            }, "p-" + i).start();
        }
    }
}
```

### 1.5 ReentrantLock 公平锁

使用`synchronized`时,锁是非公平锁,而使用ReentrantLoct时可以通过构造方法指定锁为**公平锁**或者**非公平锁**.

``` java
public ReentrantLock(boolean fair) 
```



> 公平锁: 等待时间越长越早被唤醒 
>
> 非公平锁: 唤醒的线程与该线程的等待时间无关



## 2 ReentrantLock 的实现

**ReentrantLock的lock方法:**

```java
 public void lock() {
        sync.lock();
 }
```

**ReentrantLock的unlock方法:**

```
public void unlock() {
    sync.release(1);
}
```

**sync变量:**

```
 private final Sync sync;
 abstract static class Sync extends AbstractQueuedSynchronizer{
     ...
 }
```

从上的代码可以看出 ReentrantLock 是通过 `AbstractQueuedSynchronizer`的子类来实现的.

### AQS(AbstractQueuedSynchronizer)

在ReentrantLock中对应的sync有两个实现`NonfairSync`和`FairSync` 对应非公平锁和公平锁.

###  lock方法

#### 得到锁,执行线程

`NonfairSync`中的`lock`方法

```java
       final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

在调用lock方法先使用CAS修改`AbstractQueuedSynchronizer`的 `state` 值 如果修改成功,当前线程设置当前线程为执行线程.

#### 未获取到锁 进入等待队列

如果设置失败 调用 `AbstractQueuedSynchronizer` 的 `acquire` 将当前线程放入等待队列,然后让当前线程进入中断状态.

 **`AbstractQueuedSynchronizer` 的 `acquire` 方法:**

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire`方法会先调用子类的`tryAcquire`方法, 如果这个方法返回false才会执行后面的操作. 

 使用`addWaiter`将线程封装进Node对象,添加到`AbstractQueuedSynchronizer`的Node队列中



**unlock方法**

调用了 `AbstractQueuedSynchronizer`的`release`方法, 从等待队列中获取一个线程使用`Unsafe`对象的`unpark`方法唤醒.