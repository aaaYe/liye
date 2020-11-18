# **1、volatile**

​	volatile是java虚拟机提供的轻量级同步机制，特点是1、可见性，2、禁止指令重拍。底层主要是通过内存屏障来实现的。
​	不保证原子性。原子性需要用CAS。

# **2、CAS 解决原子性**

## 2.1 概念：

​	compare and swap。
​	原子类atomicInteger。
​	底层原理：自旋锁、unsafe。  
​	unsafe类提供了硬件级别的院子操作。通过比较内存位置和预期原值，来决定是否更新值。

## 2.2 CAS源码分析

```java
/**

 * Atomically exchanges the given value with the current value of
 * a field or array element within the given object <code>o</code>
 * at the given <code>offset</code>.
   *
 * @param o object/array to update the field/element in
 * @param offset field/element offset
 * @param newValue new value
 * @return the previous value
 * @since 1.8
   */
   public final int getAndSetInt(Object o, long offset, int newValue) {
   int v;
   do {
   	v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, newValue));
   return v;
   }
```

缺点：1、循环开销大；2、只能保证一个共享变量的原子操作；3、ABA问题？

## **2.3 ABA问题？原子更新引用？**

**ABA问题**：在多线程环境中，使用lock-free的CAS时，如果一个线程对变量修改了2次，第2次修改后的值和第一次修改前的值相同，就会出现ABA问题。例子：

小明在提款机提款，

线程1：获取当前值100，期望更新50，

线程2：获取当前值100，期望更新50，

线程1执行成功后，线程2某种原因block了，这时某人给小明汇款50，

线程3（汇款线程）：获取当前值50，期望更新100，

线程2恢复后，获取到的也是100，compare之后，继续更新余额为50！！！

此时，期望余额应该是100（100-50+50），实际上变为50（100-50+50-50）。

这就是ABA问题。

**原子引用解决ABA问题**=理解原子引用+引入版本号（时间戳）

原子引用demo

```java
package com.ly.thread;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

import java.util.concurrent.atomic.AtomicReference;
@Getter
@Setter
@ToString
@AllArgsConstructor
class User {
    String username;
    int age;

}

public class AtomicReferenceDemo {

    public static void main(String[] args) {
        User z3 = new User("z3",22);
        User l4 = new User("l4",25);

        AtomicReference<Object> atomicReference = new AtomicReference<>();
        atomicReference.set(z3);

        System.out.println(atomicReference.compareAndSet(z3,l4)+"\t"+atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(z3,l4)+"\t"+atomicReference.get().toString());
    }
}
```

ABA问题解决demo

```java
package com.ly.thread;


import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * ABA问题的解决方式
 */
public class ABADemo {
    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);

        }, "t1").start();

        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2020) + "\t" + atomicReference.get());
        }, "t2").start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("==============以下解决ABA问题=============");

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号" + stamp);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第二次版本号" + atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第三次版本号" + atomicStampedReference.getStamp());
        }, "t3").start();


        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第一次版本号" + stamp);
            //暂停3秒t4线程，保证上面的t3线程完成了一次ABA操作
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean b = atomicStampedReference.compareAndSet(100, 2020, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName() + "\t修改结果:" + b + "\t当前最新实际版本号：" + atomicStampedReference.getStamp() + "\t当前实际最新值为：" + atomicStampedReference.getReference());
        }, "t4").start();
    }
}
```

