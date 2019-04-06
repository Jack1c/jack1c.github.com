---
layout: post
title: java的volatile关键字
toc: true
cover: /img/cover/why-is-binary.png
tags: ['java','并发']
category: Java
---

# java的volatile关键字

在java多线程编程中 volatile关键字有非常重要的作用,使用 volatile修饰的变量在多线程访问时可以保证变量的**可见性**.

---

###  什么是可见性?

**代码实例:**

```java
import java.util.concurrent.TimeUnit;

public class TestVolatile {
    public int i = 0;
    private static boolean flag = false;
    private static Object lock = new Object();

    private static Boolean getFlag() {
        return flag;
    }

    public static void main(String[] args) throws InterruptedException {
        TestVolatile testVolatile = new TestVolatile();
        Thread t1 = new Thread() {
            @Override
            public void run() {
                while (!getFlag()) {
                    testVolatile.i++;
                }
                System.out.println(testVolatile.i++);
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(1);
        flag = true;
    }
}
```

在上面的代码中执行的结果是 t1线程会一直执行下去, 不会因为主线程修改的flag的值而退出循环.

当一个线程修改一个变量的值,对应其他线程获取到的还是原来的值,这个值对于其他线程就是不可见.一个变量一个线程修改了它的值,其他线程在获取时能否获取到最新值称为 **可见性**.

### 为什么会产生可见性的问题?

CPU运行速度非常快, 而内存读写速度对于CPU的运行速度比较慢,为了解决这一矛盾CPU和内存之间加入了**高速缓存**.

CPU读取缓存和内存的速度差异:

+ 一次主内存的访问通常在几十到几百个时钟周期
+ 一次L1高速缓存的读写只需要1~2个时钟周期
+ 一次L2高速缓存的读写也只需要数十个时钟周期

由于访问速度的差异现代CPU基本都不会直接访问主存,而是通过**CPU缓存** ,CPU缓存是位于CPU和内存之间的临时存储器, 它的容量比较小,访问速度比较快. 使用在CPU缓存对内存中的一些数据进行缓存,当CPU访问这些数据时就比较快了. 

在执行代码时,每个CPU内核都有自己的缓存, 如果一个CPU内核修改了变量的值,而且还没有进行同步操作,那么其他的CPU内核获取这个值的时候 就不能获取到最新的值.

### 如何保证变量的可见性?

- 使用`sychronized`控制变量的访问

  ```java
  import java.util.concurrent.TimeUnit;
  
  public class TestVolatile {
      public int i = 0;
      private static boolean flag = false;
      private static Object lock = new Object();
  
      private static Boolean getFlag() {
          synchronized (lock){
              return flag;
          }
      }
  
      public static void main(String[] args) throws InterruptedException {
          TestVolatile testVolatile = new TestVolatile();
          Thread t1 = new Thread() {
              @Override
              public void run() {
                  while (!getFlag()) {
                      testVolatile.i++;
                   }
                  System.out.println(testVolatile.i++);
              }
          };
          t1.start();
          TimeUnit.SECONDS.sleep(1);
          synchronized (lock){
              flag = true;
          }
      }
  }
  ```

  使用synchronized 能控制多个线程访问一个变量,保证其可见性.

+  使用`volatile`关键字修饰变量

```java
import java.util.concurrent.TimeUnit;

public class TestVolatile {
    public int i = 0;
    private volatile static boolean flag = false;
    private static Object lock = new Object();

    private static Boolean getFlag() {
        return flag;
    }

    public static void main(String[] args) throws InterruptedException {
        TestVolatile testVolatile = new TestVolatile();
        Thread t1 = new Thread() {
            @Override
            public void run() {
                while (!getFlag()) {
                    testVolatile.i++;
                }
                System.out.println(testVolatile.i++);
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(1);
        flag = true;
    }
}
```

使用 `volatile`关键字修饰的变量, java虚拟机中保证其缓存的一致性,从而保证其对于线程的可见性.
使用  



### 从汇编代码比较有没有volatile关键字的区别

使用hsdis查看汇编代码

代码实例:

```java
public class LazySingleton {
    private static volatile LazySingleton instance = null;
    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        
        return instance;
    }
    
    public static void main(String[] args) {
        LazySingleton.getInstance();
    }
    
}
```



打印汇编代码:

```shell
 java -server -Xcomp -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -XX:CompileCommand=compileonly,*LazySingleton.getInstance
```

结果:

```assembly
Decoding compiled method 0x000000010ce1de50:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton'
  #           [sp+0x40]  (sp of caller)
  0x000000010ce1dfc0: mov    %eax,-0x14000(%rsp)
  0x000000010ce1dfc7: push   %rbp
  0x000000010ce1dfc8: sub    $0x30,%rsp
  0x000000010ce1dfcc: movabs $0x10a47a778,%rdx  ;   {metadata(method data for {method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010ce1dfd6: mov    0xdc(%rdx),%esi
  0x000000010ce1dfdc: add    $0x8,%esi
  0x000000010ce1dfdf: mov    %esi,0xdc(%rdx)
  0x000000010ce1dfe5: movabs $0x10a47a4d0,%rdx  ;   {metadata({method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010ce1dfef: and    $0x0,%esi
  0x000000010ce1dff2: cmp    $0x0,%esi
  0x000000010ce1dff5: je     0x000000010ce1e0fa
  0x000000010ce1dffb: movabs $0x7957d4a60,%rdx  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce1e005: mov    0x68(%rdx),%edx
  0x000000010ce1e008: shl    $0x3,%rdx          ;*getstatic instance
                                                ; - LazySingleton::getInstance@0 (line 6)

  0x000000010ce1e00c: cmp    $0x0,%rdx
  0x000000010ce1e010: movabs $0x10a47a778,%rdx  ;   {metadata(method data for {method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010ce1e01a: movabs $0x108,%rsi
  0x000000010ce1e024: jne    0x000000010ce1e034
  0x000000010ce1e02a: movabs $0x118,%rsi
  0x000000010ce1e034: mov    (%rdx,%rsi,1),%rdi
  0x000000010ce1e038: lea    0x1(%rdi),%rdi
  0x000000010ce1e03c: mov    %rdi,(%rdx,%rsi,1)
  0x000000010ce1e040: jne    0x000000010ce1e0dd  ;*ifnonnull
                                                ; - LazySingleton::getInstance@3 (line 6)

  0x000000010ce1e046: movabs $0x7c0061aa0,%rdx  ;   {metadata('LazySingleton')}
  0x000000010ce1e050: mov    0x60(%r15),%rax
  0x000000010ce1e054: lea    0x10(%rax),%rdi
  0x000000010ce1e058: cmp    0x70(%r15),%rdi
  0x000000010ce1e05c: ja     0x000000010ce1e111
  0x000000010ce1e062: mov    %rdi,0x60(%r15)
  0x000000010ce1e066: mov    0xa8(%rdx),%rcx
  0x000000010ce1e06d: mov    %rcx,(%rax)
  0x000000010ce1e070: mov    %rdx,%rcx
  0x000000010ce1e073: shr    $0x3,%rcx
  0x000000010ce1e077: mov    %ecx,0x8(%rax)
  0x000000010ce1e07a: xor    %rcx,%rcx
  0x000000010ce1e07d: mov    %ecx,0xc(%rax)
  0x000000010ce1e080: xor    %rcx,%rcx          ;*new  ; - LazySingleton::getInstance@6 (line 7)

  0x000000010ce1e083: mov    %rax,%rsi
  0x000000010ce1e086: movabs $0x10a47a778,%rdi  ;   {metadata(method data for {method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010ce1e090: addq   $0x1,0x128(%rdi)
  0x000000010ce1e098: mov    %rax,%rsi          ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)

  0x000000010ce1e09b: mov    %rax,0x20(%rsp)
  0x000000010ce1e0a0: nop
  0x000000010ce1e0a1: nop
  0x000000010ce1e0a2: nop
  0x000000010ce1e0a3: nop
  0x000000010ce1e0a4: nop
  0x000000010ce1e0a5: nop
  0x000000010ce1e0a6: nop
  0x000000010ce1e0a7: callq  0x000000010cd5a020  ; OopMap{[32]=Oop off=236}
                                                ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)
                                                ;   {optimized virtual_call}
  0x000000010ce1e0ac: movabs $0x7957d4a60,%rax  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce1e0b6: mov    0x20(%rsp),%rsi
  0x000000010ce1e0bb: mov    %rsi,%r10
  0x000000010ce1e0be: shr    $0x3,%r10
  0x000000010ce1e0c2: mov    %r10d,0x68(%rax)
  0x000000010ce1e0c6: shr    $0x9,%rax
  0x000000010ce1e0ca: movabs $0x105bc0000,%rsi
  0x000000010ce1e0d4: movb   $0x0,(%rax,%rsi,1)
  0x000000010ce1e0d8: lock addl $0x0,(%rsp)     ;*putstatic instance
                                                ; - LazySingleton::getInstance@13 (line 7)

  0x000000010ce1e0dd: movabs $0x7957d4a60,%rax  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce1e0e7: mov    0x68(%rax),%eax
  0x000000010ce1e0ea: shl    $0x3,%rax          ;*getstatic instance
                                                ; - LazySingleton::getInstance@16 (line 10)

  0x000000010ce1e0ee: add    $0x30,%rsp
  0x000000010ce1e0f2: pop    %rbp
  0x000000010ce1e0f3: test   %eax,-0x454fff9(%rip)        # 0x00000001088ce100
                                                ;   {poll_return}
  0x000000010ce1e0f9: retq   
  0x000000010ce1e0fa: mov    %rdx,0x8(%rsp)
  0x000000010ce1e0ff: movq   $0xffffffffffffffff,(%rsp)
  0x000000010ce1e107: callq  0x000000010ce14960  ; OopMap{off=332}
                                                ;*synchronization entry
                                                ; - LazySingleton::getInstance@-1 (line 6)
                                                ;   {runtime_call}
  0x000000010ce1e10c: jmpq   0x000000010ce1dffb
  0x000000010ce1e111: mov    %rdx,%rdx
  0x000000010ce1e114: callq  0x000000010cd83b20  ; OopMap{off=345}
                                                ;*new  ; - LazySingleton::getInstance@6 (line 7)
                                                ;   {runtime_call}
  0x000000010ce1e119: jmpq   0x000000010ce1e083
  0x000000010ce1e11e: nop
  0x000000010ce1e11f: nop
  0x000000010ce1e120: mov    0x2a8(%r15),%rax
  0x000000010ce1e127: movabs $0x0,%r10
  0x000000010ce1e131: mov    %r10,0x2a8(%r15)
  0x000000010ce1e138: movabs $0x0,%r10
  0x000000010ce1e142: mov    %r10,0x2b0(%r15)
  0x000000010ce1e149: add    $0x30,%rsp
  0x000000010ce1e14d: pop    %rbp
  0x000000010ce1e14e: jmpq   0x000000010cd84fe0  ;   {runtime_call}
  0x000000010ce1e153: hlt    
  0x000000010ce1e154: hlt    
  0x000000010ce1e155: hlt    
  0x000000010ce1e156: hlt    
  0x000000010ce1e157: hlt    
  0x000000010ce1e158: hlt    
  0x000000010ce1e159: hlt    
  0x000000010ce1e15a: hlt    
  0x000000010ce1e15b: hlt    
  0x000000010ce1e15c: hlt    
  0x000000010ce1e15d: hlt    
  0x000000010ce1e15e: hlt    
  0x000000010ce1e15f: hlt    
[Stub Code]
  0x000000010ce1e160: nop                       ;   {no_reloc}
  0x000000010ce1e161: nop
  0x000000010ce1e162: nop
  0x000000010ce1e163: nop
  0x000000010ce1e164: nop
  0x000000010ce1e165: movabs $0x0,%rbx          ;   {static_stub}
  0x000000010ce1e16f: jmpq   0x000000010ce1e16f  ;   {runtime_call}
[Exception Handler]
  0x000000010ce1e174: callq  0x000000010ce122e0  ;   {runtime_call}
  0x000000010ce1e179: mov    %rsp,-0x28(%rsp)
  0x000000010ce1e17e: sub    $0x80,%rsp
  0x000000010ce1e185: mov    %rax,0x78(%rsp)
  0x000000010ce1e18a: mov    %rcx,0x70(%rsp)
  0x000000010ce1e18f: mov    %rdx,0x68(%rsp)
  0x000000010ce1e194: mov    %rbx,0x60(%rsp)
  0x000000010ce1e199: mov    %rbp,0x50(%rsp)
  0x000000010ce1e19e: mov    %rsi,0x48(%rsp)
  0x000000010ce1e1a3: mov    %rdi,0x40(%rsp)
  0x000000010ce1e1a8: mov    %r8,0x38(%rsp)
  0x000000010ce1e1ad: mov    %r9,0x30(%rsp)
  0x000000010ce1e1b2: mov    %r10,0x28(%rsp)
  0x000000010ce1e1b7: mov    %r11,0x20(%rsp)
  0x000000010ce1e1bc: mov    %r12,0x18(%rsp)
  0x000000010ce1e1c1: mov    %r13,0x10(%rsp)
  0x000000010ce1e1c6: mov    %r14,0x8(%rsp)
  0x000000010ce1e1cb: mov    %r15,(%rsp)
  0x000000010ce1e1cf: movabs $0x107ec734d,%rdi  ;   {external_word}
  0x000000010ce1e1d9: movabs $0x10ce1e179,%rsi  ;   {internal_word}
  0x000000010ce1e1e3: mov    %rsp,%rdx
  0x000000010ce1e1e6: and    $0xfffffffffffffff0,%rsp
  0x000000010ce1e1ea: callq  0x0000000107cf52c8  ;   {runtime_call}
  0x000000010ce1e1ef: hlt    
[Deopt Handler Code]
  0x000000010ce1e1f0: movabs $0x10ce1e1f0,%r10  ;   {section_word}
  0x000000010ce1e1fa: push   %r10
  0x000000010ce1e1fc: jmpq   0x000000010cd5b3c0  ;   {runtime_call}
  0x000000010ce1e201: hlt    
  0x000000010ce1e202: hlt    
  0x000000010ce1e203: hlt    
  0x000000010ce1e204: hlt    
  0x000000010ce1e205: hlt    
  0x000000010ce1e206: hlt    
  0x000000010ce1e207: hlt    
Decoding compiled method 0x000000010ce21090:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x000000010a47a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton'
  #           [sp+0x20]  (sp of caller)
  0x000000010ce211e0: mov    %eax,-0x14000(%rsp)
  0x000000010ce211e7: push   %rbp
  0x000000010ce211e8: sub    $0x10,%rsp         ;*synchronization entry
                                                ; - LazySingleton::getInstance@-1 (line 6)

  0x000000010ce211ec: movabs $0x7957d4a60,%r10  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce211f6: mov    0x68(%r10),%r11d   ;*getstatic instance
                                                ; - LazySingleton::getInstance@0 (line 6)

  0x000000010ce211fa: test   %r11d,%r11d
  0x000000010ce211fd: je     0x000000010ce21220
  0x000000010ce211ff: movabs $0x7957d4a60,%r10  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce21209: mov    0x68(%r10),%r11d
  0x000000010ce2120d: mov    %r11,%rax
  0x000000010ce21210: shl    $0x3,%rax          ;*getstatic instance
                                                ; - LazySingleton::getInstance@16 (line 10)

  0x000000010ce21214: add    $0x10,%rsp
  0x000000010ce21218: pop    %rbp
  0x000000010ce21219: test   %eax,-0x455321f(%rip)        # 0x00000001088ce000
                                                ;   {poll_return}
  0x000000010ce2121f: retq   
  0x000000010ce21220: mov    0x60(%r15),%rax
  0x000000010ce21224: mov    %rax,%r10
  0x000000010ce21227: add    $0x10,%r10
  0x000000010ce2122b: cmp    0x70(%r15),%r10
  0x000000010ce2122f: jae    0x000000010ce212a5
  0x000000010ce21231: mov    %r10,0x60(%r15)
  0x000000010ce21235: prefetchw 0xc0(%r10)
  0x000000010ce2123d: mov    $0xf800c354,%r11d  ;   {metadata('LazySingleton')}
  0x000000010ce21243: movabs $0x0,%r10
  0x000000010ce2124d: lea    (%r10,%r11,8),%r10
  0x000000010ce21251: mov    0xa8(%r10),%r10
  0x000000010ce21258: mov    %r10,(%rax)
  0x000000010ce2125b: movl   $0xf800c354,0x8(%rax)  ;   {metadata('LazySingleton')}
  0x000000010ce21262: mov    %r12d,0xc(%rax)
  0x000000010ce21266: mov    %rax,%rbp          ;*new  ; - LazySingleton::getInstance@6 (line 7)

  0x000000010ce21269: mov    %rbp,%rsi
  0x000000010ce2126c: data32 xchg %ax,%ax
  0x000000010ce2126f: callq  0x000000010cd5a020  ; OopMap{rbp=Oop off=148}
                                                ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)
                                                ;   {optimized virtual_call}
  0x000000010ce21274: mov    %rbp,%r11
  0x000000010ce21277: shr    $0x3,%r11
  0x000000010ce2127b: movabs $0x7957d4a60,%r10  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010ce21285: mov    %r11d,0x68(%r10)
  0x000000010ce21289: shr    $0x9,%r10
  0x000000010ce2128d: movabs $0x105bc0000,%r11
  0x000000010ce21297: mov    %r12b,(%r11,%r10,1)
  0x000000010ce2129b: lock addl $0x0,(%rsp)     ;*putstatic instance
                                                ; - LazySingleton::getInstance@13 (line 7)

  0x000000010ce212a0: jmpq   0x000000010ce211ff
  0x000000010ce212a5: movabs $0x7c0061aa0,%rsi  ;   {metadata('LazySingleton')}
  0x000000010ce212af: callq  0x000000010cd82560  ; OopMap{off=212}
                                                ;*new  ; - LazySingleton::getInstance@6 (line 7)
                                                ;   {runtime_call}
  0x000000010ce212b4: jmp    0x000000010ce21266  ;*new
                                                ; - LazySingleton::getInstance@6 (line 7)

  0x000000010ce212b6: mov    %rax,%rsi
  0x000000010ce212b9: jmp    0x000000010ce212be
  0x000000010ce212bb: mov    %rax,%rsi          ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)

  0x000000010ce212be: add    $0x10,%rsp
  0x000000010ce212c2: pop    %rbp
  0x000000010ce212c3: jmpq   0x000000010cd852a0  ;   {runtime_call}
  0x000000010ce212c8: hlt    
  0x000000010ce212c9: hlt    
  0x000000010ce212ca: hlt    
  0x000000010ce212cb: hlt    
  0x000000010ce212cc: hlt    
  0x000000010ce212cd: hlt    
  0x000000010ce212ce: hlt    
  0x000000010ce212cf: hlt    
  0x000000010ce212d0: hlt    
  0x000000010ce212d1: hlt    
  0x000000010ce212d2: hlt    
  0x000000010ce212d3: hlt    
  0x000000010ce212d4: hlt    
  0x000000010ce212d5: hlt    
  0x000000010ce212d6: hlt    
  0x000000010ce212d7: hlt    
  0x000000010ce212d8: hlt    
  0x000000010ce212d9: hlt    
  0x000000010ce212da: hlt    
  0x000000010ce212db: hlt    
  0x000000010ce212dc: hlt    
  0x000000010ce212dd: hlt    
  0x000000010ce212de: hlt    
  0x000000010ce212df: hlt    
[Stub Code]
  0x000000010ce212e0: movabs $0x0,%rbx          ;   {no_reloc}
  0x000000010ce212ea: jmpq   0x000000010ce212ea  ;   {runtime_call}
[Exception Handler]
  0x000000010ce212ef: jmpq   0x000000010cd6b6a0  ;   {runtime_call}
[Deopt Handler Code]
  0x000000010ce212f4: callq  0x000000010ce212f9
  0x000000010ce212f9: subq   $0x5,(%rsp)
  0x000000010ce212fe: jmpq   0x000000010cd5b3c0  ;   {runtime_call}
  0x000000010ce21303: hlt    
  0x000000010ce21304: hlt    
  0x000000010ce21305: hlt    
  0x000000010ce21306: hlt    
  0x000000010ce21307: hlt    
```

将`volatile`删掉 再编译查看两次结果的区别

java代码:

```
public class LazySingleton {
    private static LazySingleton instance = null;
    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        
        return instance;
    }
    
    public static void main(String[] args) {
        LazySingleton.getInstance();
    }
    
}
```

编译后的汇编代码:

```assembly
Loaded disassembler from /Library/Java/JavaVirtualMachines/jdk1.8.0_91.jdk/Contents/Home/jre/lib/server/hsdis-amd64.dylib
Decoding compiled method 0x000000010fc16e50:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton'
  #           [sp+0x40]  (sp of caller)
  0x000000010fc16fc0: mov    %eax,-0x14000(%rsp)
  0x000000010fc16fc7: push   %rbp
  0x000000010fc16fc8: sub    $0x30,%rsp
  0x000000010fc16fcc: movabs $0x10ce7a778,%rdx  ;   {metadata(method data for {method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010fc16fd6: mov    0xdc(%rdx),%esi
  0x000000010fc16fdc: add    $0x8,%esi
  0x000000010fc16fdf: mov    %esi,0xdc(%rdx)
  0x000000010fc16fe5: movabs $0x10ce7a4d0,%rdx  ;   {metadata({method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010fc16fef: and    $0x0,%esi
  0x000000010fc16ff2: cmp    $0x0,%esi
  0x000000010fc16ff5: je     0x000000010fc170f5
  0x000000010fc16ffb: movabs $0x7957d4a60,%rdx  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc17005: mov    0x68(%rdx),%edx
  0x000000010fc17008: shl    $0x3,%rdx          ;*getstatic instance
                                                ; - LazySingleton::getInstance@0 (line 6)

  0x000000010fc1700c: cmp    $0x0,%rdx
  0x000000010fc17010: movabs $0x10ce7a778,%rdx  ;   {metadata(method data for {method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010fc1701a: movabs $0x108,%rsi
  0x000000010fc17024: jne    0x000000010fc17034
  0x000000010fc1702a: movabs $0x118,%rsi
  0x000000010fc17034: mov    (%rdx,%rsi,1),%rdi
  0x000000010fc17038: lea    0x1(%rdi),%rdi
  0x000000010fc1703c: mov    %rdi,(%rdx,%rsi,1)
  0x000000010fc17040: jne    0x000000010fc170d8  ;*ifnonnull
                                                ; - LazySingleton::getInstance@3 (line 6)

  0x000000010fc17046: movabs $0x7c0061aa0,%rdx  ;   {metadata('LazySingleton')}
  0x000000010fc17050: mov    0x60(%r15),%rax
  0x000000010fc17054: lea    0x10(%rax),%rdi
  0x000000010fc17058: cmp    0x70(%r15),%rdi
  0x000000010fc1705c: ja     0x000000010fc1710c
  0x000000010fc17062: mov    %rdi,0x60(%r15)
  0x000000010fc17066: mov    0xa8(%rdx),%rcx
  0x000000010fc1706d: mov    %rcx,(%rax)
  0x000000010fc17070: mov    %rdx,%rcx
  0x000000010fc17073: shr    $0x3,%rcx
  0x000000010fc17077: mov    %ecx,0x8(%rax)
  0x000000010fc1707a: xor    %rcx,%rcx
  0x000000010fc1707d: mov    %ecx,0xc(%rax)
  0x000000010fc17080: xor    %rcx,%rcx          ;*new  ; - LazySingleton::getInstance@6 (line 7)

  0x000000010fc17083: mov    %rax,%rsi
  0x000000010fc17086: movabs $0x10ce7a778,%rdi  ;   {metadata(method data for {method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton')}
  0x000000010fc17090: addq   $0x1,0x128(%rdi)
  0x000000010fc17098: mov    %rax,%rsi          ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)

  0x000000010fc1709b: mov    %rax,0x20(%rsp)
  0x000000010fc170a0: nop
  0x000000010fc170a1: nop
  0x000000010fc170a2: nop
  0x000000010fc170a3: nop
  0x000000010fc170a4: nop
  0x000000010fc170a5: nop
  0x000000010fc170a6: nop
  0x000000010fc170a7: callq  0x000000010fb53020  ; OopMap{[32]=Oop off=236}
                                                ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)
                                                ;   {optimized virtual_call}
  0x000000010fc170ac: movabs $0x7957d4a60,%rax  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc170b6: mov    0x20(%rsp),%rsi
  0x000000010fc170bb: mov    %rsi,%r10
  0x000000010fc170be: shr    $0x3,%r10
  0x000000010fc170c2: mov    %r10d,0x68(%rax)
  0x000000010fc170c6: shr    $0x9,%rax
  0x000000010fc170ca: movabs $0x1085c0000,%rsi
  0x000000010fc170d4: movb   $0x0,(%rax,%rsi,1)  ;*putstatic instance
                                                ; - LazySingleton::getInstance@13 (line 7)

  0x000000010fc170d8: movabs $0x7957d4a60,%rax  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc170e2: mov    0x68(%rax),%eax
  0x000000010fc170e5: shl    $0x3,%rax          ;*getstatic instance
                                                ; - LazySingleton::getInstance@16 (line 10)

  0x000000010fc170e9: add    $0x30,%rsp
  0x000000010fc170ed: pop    %rbp
  0x000000010fc170ee: test   %eax,-0x49a3ff4(%rip)        # 0x000000010b273100
                                                ;   {poll_return}
  0x000000010fc170f4: retq   
  0x000000010fc170f5: mov    %rdx,0x8(%rsp)
  0x000000010fc170fa: movq   $0xffffffffffffffff,(%rsp)
  0x000000010fc17102: callq  0x000000010fc0d960  ; OopMap{off=327}
                                                ;*synchronization entry
                                                ; - LazySingleton::getInstance@-1 (line 6)
                                                ;   {runtime_call}
  0x000000010fc17107: jmpq   0x000000010fc16ffb
  0x000000010fc1710c: mov    %rdx,%rdx
  0x000000010fc1710f: callq  0x000000010fb7cb20  ; OopMap{off=340}
                                                ;*new  ; - LazySingleton::getInstance@6 (line 7)
                                                ;   {runtime_call}
  0x000000010fc17114: jmpq   0x000000010fc17083
  0x000000010fc17119: nop
  0x000000010fc1711a: nop
  0x000000010fc1711b: mov    0x2a8(%r15),%rax
  0x000000010fc17122: movabs $0x0,%r10
  0x000000010fc1712c: mov    %r10,0x2a8(%r15)
  0x000000010fc17133: movabs $0x0,%r10
  0x000000010fc1713d: mov    %r10,0x2b0(%r15)
  0x000000010fc17144: add    $0x30,%rsp
  0x000000010fc17148: pop    %rbp
  0x000000010fc17149: jmpq   0x000000010fb7dfe0  ;   {runtime_call}
  0x000000010fc1714e: hlt    
  0x000000010fc1714f: hlt    
  0x000000010fc17150: hlt    
  0x000000010fc17151: hlt    
  0x000000010fc17152: hlt    
  0x000000010fc17153: hlt    
  0x000000010fc17154: hlt    
  0x000000010fc17155: hlt    
  0x000000010fc17156: hlt    
  0x000000010fc17157: hlt    
  0x000000010fc17158: hlt    
  0x000000010fc17159: hlt    
  0x000000010fc1715a: hlt    
  0x000000010fc1715b: hlt    
  0x000000010fc1715c: hlt    
  0x000000010fc1715d: hlt    
  0x000000010fc1715e: hlt    
  0x000000010fc1715f: hlt    
[Stub Code]
  0x000000010fc17160: nop                       ;   {no_reloc}
  0x000000010fc17161: nop
  0x000000010fc17162: nop
  0x000000010fc17163: nop
  0x000000010fc17164: nop
  0x000000010fc17165: movabs $0x0,%rbx          ;   {static_stub}
  0x000000010fc1716f: jmpq   0x000000010fc1716f  ;   {runtime_call}
[Exception Handler]
  0x000000010fc17174: callq  0x000000010fc0b2e0  ;   {runtime_call}
  0x000000010fc17179: mov    %rsp,-0x28(%rsp)
  0x000000010fc1717e: sub    $0x80,%rsp
  0x000000010fc17185: mov    %rax,0x78(%rsp)
  0x000000010fc1718a: mov    %rcx,0x70(%rsp)
  0x000000010fc1718f: mov    %rdx,0x68(%rsp)
  0x000000010fc17194: mov    %rbx,0x60(%rsp)
  0x000000010fc17199: mov    %rbp,0x50(%rsp)
  0x000000010fc1719e: mov    %rsi,0x48(%rsp)
  0x000000010fc171a3: mov    %rdi,0x40(%rsp)
  0x000000010fc171a8: mov    %r8,0x38(%rsp)
  0x000000010fc171ad: mov    %r9,0x30(%rsp)
  0x000000010fc171b2: mov    %r10,0x28(%rsp)
  0x000000010fc171b7: mov    %r11,0x20(%rsp)
  0x000000010fc171bc: mov    %r12,0x18(%rsp)
  0x000000010fc171c1: mov    %r13,0x10(%rsp)
  0x000000010fc171c6: mov    %r14,0x8(%rsp)
  0x000000010fc171cb: mov    %r15,(%rsp)
  0x000000010fc171cf: movabs $0x10a86c34d,%rdi  ;   {external_word}
  0x000000010fc171d9: movabs $0x10fc17179,%rsi  ;   {internal_word}
  0x000000010fc171e3: mov    %rsp,%rdx
  0x000000010fc171e6: and    $0xfffffffffffffff0,%rsp
  0x000000010fc171ea: callq  0x000000010a69a2c8  ;   {runtime_call}
  0x000000010fc171ef: hlt    
[Deopt Handler Code]
  0x000000010fc171f0: movabs $0x10fc171f0,%r10  ;   {section_word}
  0x000000010fc171fa: push   %r10
  0x000000010fc171fc: jmpq   0x000000010fb543c0  ;   {runtime_call}
  0x000000010fc17201: hlt    
  0x000000010fc17202: hlt    
  0x000000010fc17203: hlt    
  0x000000010fc17204: hlt    
  0x000000010fc17205: hlt    
  0x000000010fc17206: hlt    
  0x000000010fc17207: hlt    
Decoding compiled method 0x000000010fc1a050:
Code:
[Entry Point]
[Verified Entry Point]
[Constants]
  # {method} {0x000000010ce7a4d0} 'getInstance' '()LLazySingleton;' in 'LazySingleton'
  #           [sp+0x20]  (sp of caller)
  0x000000010fc1a1a0: mov    %eax,-0x14000(%rsp)
  0x000000010fc1a1a7: push   %rbp
  0x000000010fc1a1a8: sub    $0x10,%rsp         ;*synchronization entry
                                                ; - LazySingleton::getInstance@-1 (line 6)

  0x000000010fc1a1ac: movabs $0x7957d4a60,%r10  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc1a1b6: mov    0x68(%r10),%r11d   ;*getstatic instance
                                                ; - LazySingleton::getInstance@0 (line 6)

  0x000000010fc1a1ba: test   %r11d,%r11d
  0x000000010fc1a1bd: je     0x000000010fc1a1d2  ;*ifnonnull
                                                ; - LazySingleton::getInstance@3 (line 6)

  0x000000010fc1a1bf: mov    %r11,%rax
  0x000000010fc1a1c2: shl    $0x3,%rax
  0x000000010fc1a1c6: add    $0x10,%rsp
  0x000000010fc1a1ca: pop    %rbp
  0x000000010fc1a1cb: test   %eax,-0x49a71d1(%rip)        # 0x000000010b273000
                                                ;   {poll_return}
  0x000000010fc1a1d1: retq   
  0x000000010fc1a1d2: mov    0x60(%r15),%rax
  0x000000010fc1a1d6: mov    %rax,%r10
  0x000000010fc1a1d9: add    $0x10,%r10
  0x000000010fc1a1dd: cmp    0x70(%r15),%r10
  0x000000010fc1a1e1: jae    0x000000010fc1a255
  0x000000010fc1a1e3: mov    %r10,0x60(%r15)
  0x000000010fc1a1e7: prefetchw 0xc0(%r10)
  0x000000010fc1a1ef: mov    $0xf800c354,%r10d  ;   {metadata('LazySingleton')}
  0x000000010fc1a1f5: shl    $0x3,%r10
  0x000000010fc1a1f9: mov    0xa8(%r10),%r10
  0x000000010fc1a200: mov    %r10,(%rax)
  0x000000010fc1a203: movl   $0xf800c354,0x8(%rax)  ;   {metadata('LazySingleton')}
  0x000000010fc1a20a: mov    %r12d,0xc(%rax)
  0x000000010fc1a20e: mov    %rax,%rbp          ;*new  ; - LazySingleton::getInstance@6 (line 7)

  0x000000010fc1a211: mov    %rbp,%rsi
  0x000000010fc1a214: data32 xchg %ax,%ax
  0x000000010fc1a217: callq  0x000000010fb53020  ; OopMap{rbp=Oop off=124}
                                                ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)
                                                ;   {optimized virtual_call}
  0x000000010fc1a21c: movabs $0x7957d4a60,%r10  ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc1a226: mov    %rbp,%r11
  0x000000010fc1a229: shr    $0x3,%r11
  0x000000010fc1a22d: movabs $0x7957d4a60,%r8   ;   {oop(a 'java/lang/Class' = 'LazySingleton')}
  0x000000010fc1a237: mov    %r11d,0x68(%r8)
  0x000000010fc1a23b: shr    $0x9,%r10
  0x000000010fc1a23f: movabs $0x1085c0000,%r11
  0x000000010fc1a249: mov    %r12b,(%r11,%r10,1)  ;*putstatic instance
                                                ; - LazySingleton::getInstance@13 (line 7)

  0x000000010fc1a24d: mov    %rbp,%rax
  0x000000010fc1a250: jmpq   0x000000010fc1a1c6
  0x000000010fc1a255: movabs $0x7c0061aa0,%rsi  ;   {metadata('LazySingleton')}
  0x000000010fc1a25f: callq  0x000000010fb7b560  ; OopMap{off=196}
                                                ;*new  ; - LazySingleton::getInstance@6 (line 7)
                                                ;   {runtime_call}
  0x000000010fc1a264: jmp    0x000000010fc1a20e  ;*new
                                                ; - LazySingleton::getInstance@6 (line 7)

  0x000000010fc1a266: mov    %rax,%rsi
  0x000000010fc1a269: jmp    0x000000010fc1a26e
  0x000000010fc1a26b: mov    %rax,%rsi          ;*invokespecial <init>
                                                ; - LazySingleton::getInstance@10 (line 7)

  0x000000010fc1a26e: add    $0x10,%rsp
  0x000000010fc1a272: pop    %rbp
  0x000000010fc1a273: jmpq   0x000000010fb7e2a0  ;   {runtime_call}
  0x000000010fc1a278: hlt    
  0x000000010fc1a279: hlt    
  0x000000010fc1a27a: hlt    
  0x000000010fc1a27b: hlt    
  0x000000010fc1a27c: hlt    
  0x000000010fc1a27d: hlt    
  0x000000010fc1a27e: hlt    
  0x000000010fc1a27f: hlt    
[Stub Code]
  0x000000010fc1a280: movabs $0x0,%rbx          ;   {no_reloc}
  0x000000010fc1a28a: jmpq   0x000000010fc1a28a  ;   {runtime_call}
[Exception Handler]
  0x000000010fc1a28f: jmpq   0x000000010fb646a0  ;   {runtime_call}
[Deopt Handler Code]
  0x000000010fc1a294: callq  0x000000010fc1a299
  0x000000010fc1a299: subq   $0x5,(%rsp)
  0x000000010fc1a29e: jmpq   0x000000010fb543c0  ;   {runtime_call}
```

比较两次的汇编代码可以看出加了volatile的变量在汇编的77行不同, 加了一个`lock`指令

```assembly
0x000000010ce1e0d8: lock addl $0x0,(%rsp)     ;*putstatic instance
                                                ; - LazySingleton::getInstance@13 (line 7)
```



### lock指令的作用

- 锁总线,对于其他CPU对内存的读写都进行阻塞.后来的CPU都都采用锁缓存代替总线.因为锁总线的开销较大,在锁总线的过程中其他CPU无法访问内存
- lock后的写操作会 回写已修改的数据,同时让其他CPU的缓存行失败,从而从内存中加载新的数据.
- 能完成类似内存屏障的功能, 阻止屏障两边的指令重排序



### 由lock指令回看volatile变量读写

![线程-内存](https://raw.githubusercontent.com/Jack1c/Jack1c.github.io/master/_posts/media/%E7%BA%BF%E7%A8%8B-%E5%86%85%E5%AD%98.png)


工作内存Work Memory其实就是对CPU寄存器和高速缓存的抽象，或者说每个线程的工作内存也可以简单理解为CPU寄存器和高速缓存。

那么当写两条线程Thread-A与Threab-B同时操作主存中的一个volatile变量i时, Thread-A写了变量i，那么：

- Thread-A发出LOCK#指令
- 发出的LOCK#指令锁总线（或锁缓存行），同时让Thread-B高速缓存中的缓存行内容失效
- Thread-A向主存回写最新修改的i

Thread-B读取变量i，那么：

- Thread-B发现对应地址的缓存行被锁了，等待锁的释放，缓存一致性协议会保证它读取到最新的值

由此可以看出，volatile关键字的读和普通变量的读取相比基本没差别，差别主要还是在变量的写操作上。

 

参考: 

 [就是要你懂Java中volatile关键字实现原理](https://www.cnblogs.com/xrq730/p/7048693.html)