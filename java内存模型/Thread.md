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

**缺点**：1、循环开销大；2、只能保证一个共享变量的原子操作；3、ABA问题？

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

# 3、集合类线程不安全问题

## 3.1 ArrayList是线程安全的，给出线程安全的解决方案？

ArrayList经常出现的异常 java.util.ConcurrentModificationException（并发修改异常）

解决方案：

Vector

Collections.synchronizedList(new ArrayList<>())

CopyOnWriteArrayList

CopyOnWriteArrayList的源码分析：读写分离，写时复制。

```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 3.2 Set

HashSet的底层是HashMap。为什么HashSet的add方法不用写Value呢？value是一个常量。

```java
private static final Object PRESENT = new Object();
/**
 * Adds the specified element to this set if it is not already present.
 * More formally, adds the specified element <tt>e</tt> to this set if
 * this set contains no element <tt>e2</tt> such that
 * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
 * If this set already contains the element, the call leaves the set
 * unchanged and returns <tt>false</tt>.
 *
 * @param e element to be added to this set
 * @return <tt>true</tt> if this set did not already contain the specified
 * element
 */
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

## 3.3 Map

HashMap底层数据结构：初始长度为16（方便位运算），数组，链表，链表长度超过8时有红黑树。

# 4、锁

## 4.1 公平锁和非公平锁

公平锁：多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远是队列的第一位才能获得锁。

- 优点：所有线程都能得到资源，不会饿死在线程中。
- 缺点：吞吐量下降，队列里除了第一个线程，其他的线程都会阻塞，cpu唤醒线程的开销大。

非公平锁：多个线程去获取锁的时候，会直接去获取，获取不到在进入队列排队，如果能获取到，就直接加塞。

- ReentrantLock（默认非公平），synchronized（非公平）
- 优点：减少CPU唤醒线程的开销，整体的吞吐量会高点，CPU也不必唤醒所有线程，会减少线程唤醒的数量。
- 缺点：导致线程中的某一线程一直获取不到锁饿死。

## 4.2 可重入锁（递归锁）

指的是同一线程外层函数获得锁后，内存函数仍然能获取该锁的代码。在同一线程在外层方法获取锁的时候，在内层方法会自动获得锁。也即是说，线程可以进入任何一个它已经拥有的锁所同步着的代码块。

ReentrantLock、synchronized

最大的作用是避免死锁。

## 4.3 自旋锁（spinlock）

指当一个线程在获取锁的时候，如果锁已经被其他线程获取，那么该线程将循环等待，然后不断的判断是否能够被成功获取，直到获取到锁才会退出循环。

缺点：获取锁的线程会一直处于活跃状态，但是并没有执行任务，使用这种锁会造成busy-waiting。

demo：CAS原子类。

```java
package com.ly.thread;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

/**
 * @Author liye
 * @Date 2020/11/22 20:07
 */

public class SpinLockDemo {

    //原子引用线程
    AtomicReference<Thread> atomicReference = new AtomicReference();

    public void myLock() {
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\t come in ovo");
        while (!atomicReference.compareAndSet(null, thread)) {

        }
    }

    public void myUnLock(){
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread,null);
        System.out.println(Thread.currentThread().getName()+"\t invoked lock");
    }

    public static void main(String[] args) {
        SpinLockDemo spinLockDemo = new SpinLockDemo();
        new Thread(() -> {
            spinLockDemo.myLock();
            try {
                TimeUnit.SECONDS.sleep(5);
                spinLockDemo.myUnLock();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"A").start();

        new Thread(() -> {
            spinLockDemo.myLock();
            spinLockDemo.myUnLock();
        },"B").start();

    }
}
```

## 4.4  独占锁（写锁）/共享锁（读锁）/互斥锁

- 独占锁：该锁只能被一个线程占用。ReentrantLock、synchronized
- 共享锁：该锁可以被多个线程所持有。
- 对ReentrantReadWriteLock其读锁是共享锁，其写锁是独占锁。
- 读锁的共享锁可保证并发读是高效的，读写，写读，写写的过程是互斥的。

### 读写锁ReentranReadWriteLock:

```java
package com.ly.thread;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class MyCache {
    private volatile Map<String, Object> map = new HashMap();
    private ReentrantReadWriteLock rwlock = new ReentrantReadWriteLock();

    public void put(String key, Object value) {

        rwlock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入" + key);
            try {
                TimeUnit.MICROSECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t 写入完成");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rwlock.writeLock().unlock();
        }
    }

    public void get(String key) {
        rwlock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取" + key);

            try {
                TimeUnit.MICROSECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t 读取完成：" + result);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rwlock.readLock().unlock();
        }

    }

    public void clear(){
        map.clear();
    }
}

/**
 * 多线程同时读一个资源类没有问题，所以为了满足并发量，读取资源共享应该可以同时进行
 * 但是
 * 如果有一个线程想去写共享资源，就不应该有其他线程可以再进行读写
 * 读-读共存
 * 读-写，写-写不共存
 */
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache myCache = new MyCache();
        for (int i = 0; i < 5; i++) {
            final int temp = i;
            new Thread(() -> {
                myCache.put(temp + "", temp + "");
            }, String.valueOf(i)).start();
        }

        for (int i = 0; i < 5; i++) {
            final int temp = i;
            new Thread(() -> {
                myCache.get(temp + "");
            }, String.valueOf(i)).start();
        }

    }
}
```

### CountDownLatch闭锁：计数器。

让一些线程阻塞知道另一线程完成一系列操作之后才被唤醒。

CountDownLatch主要有两个方法，当一个或多个线程调用await方法时，调用线程会被阻塞。其他线程调用countDown方法会将计数器减1（调用countDown方法的线程不会被阻塞），当计数器的值变为0时，因调用await方法被阻塞的线程会被唤醒，继续执行。

```java
package com.ly.thread;


import java.util.concurrent.CountDownLatch;

enum CountryEunm {
    ONE(1, "齐国"), TWO(2, "楚国"), THREE(3, "燕国"), FOUR(4, "赵国"), FIVE(5, "魏国"), SIX(6, "韩国");

    private Integer retCode;

    private String retMessage;

    public Integer getRetCode() {
        return retCode;
    }

    public String getRetMessage() {
        return retMessage;
    }

    CountryEunm(Integer retCode, String retMessage) {
        this.retCode = retCode;
        this.retMessage = retMessage;
    }

    public static CountryEunm foreach_CountryEunm(int index) {
        CountryEunm[] values = CountryEunm.values();
        for (CountryEunm element : values) {
            if (index == element.getRetCode()){
                return element;
            }
        }
        return null;
    }
}

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {

        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 亡");
                countDownLatch.countDown();
            }, CountryEunm.foreach_CountryEunm(i).getRetMessage()).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + "\t 秦灭六国");

    }

    private static void closed() throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t leave");
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + "\t closed");
    }
}
```

### CyclicBarrier:与CountDownLatch相反

```java
package com.ly.thread;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,()->{System.out.println("**************召唤神龙");});
        for (int i = 1; i <= 7; i++) {
            final  int temp = i;
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName()+"\t 收集到了第"+temp+"颗龙珠");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            },String.valueOf(i)).start();
        }

    }
}
```

### Semaphore:多个线程抢占多个资源

```java
package com.ly.thread;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);
        for (int i = 1; i <=6; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName()+"\t抢到车位");
                    TimeUnit.SECONDS.sleep(3);
                    System.out.println(Thread.currentThread().getName()+"\t停车3秒后离开");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release();
                }
            },String.valueOf(i)).start();
        }
    }
}
```

# 5、阻塞队列

## 5.1概念：

阻塞队列，顾名思义，首先是一个队列，而一个阻塞队列在数据结构中所起到的作用大致是这样：

<img src="D:\workspace\myfile\liye\image\阻塞队列.png" alt="阻塞队列" style="zoom:60%;" />

当阻塞队列为空的时候，take获取元素操作会被阻塞。

当阻塞队列是满的时候，put添加元素操作会被阻塞。

为什么要使用阻塞队列？

​	在对线程领域：所谓阻塞，在某些情况会挂起，一旦条件满足，被挂起的线程又会被自动唤醒。

为什么要使用BlockingQueue?

好处是我们不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程，因为这一切BlockingQueue一手包办了。

在concurrent包发布以前，在多线程环境下，我们每个程序员都必须去自己控制这些细节，尤其还要兼顾效率和线程安全，而这会给我们程序带来不小的复杂度。

## 5.2 阻塞队列种类：

- ArrayBlockingQueue:由数组结构组成的有界阻塞队列。
- LinkedBlockingQueue:由链表结构组成的有界阻塞队列(但界的大小默认为Integer.MAX_VALUE)。
- PriorityBlockingQueue:支持优先级排序的无界阻塞队列。
- DelayQueue:使用优先级队列实现的延迟无界阻塞队列。
- SyschronousQueue:不存储元素的阻塞队列，也即单个元素的队列。
- LinkedTransferQueue:由链表组成的无界阻塞队列。
- LinkedBlockingDeque:由链表组成的双向阻塞队列。

![阻塞队列成员](D:\workspace\myfile\liye\image\阻塞队列成员.png)

## 5.3 阻塞队列的核心方法：

![阻塞队列的核心方法](D:\workspace\myfile\liye\image\阻塞队列的核心方法.png)

1. 抛出异常：当线程阻塞队列满的时候，再往队列里add元素，会抛出IllegalStateException("Queue full")异常。当队列为空时，再从队列里remove元素，会抛出NoSuchElementException异常。
2. 返回特殊值：offer(e)方法会返回是否成功，true或false；poll()方法从队列里移除一个元素，队列为空则返回null。
3. 一直阻塞：当阻塞队列满时，如果生产者线程往阻塞队列里put(e)元素，队列会一直阻塞生产者线程，直到拿到数据，或者响应中断退出。当队列为空时，消费者线程试图从阻塞队列里take()线程，队列也会阻塞消费者队列，直到队列可用。
4. 超时退出：当队列满时，队列会阻塞生产者线程一段时间，如果超过一段时间，生产者线程就会退出。

## 5.4 传统阻塞队列生产者消费者模式

```java
package com.ly.thread;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

class ShareData {
    private int number = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void increment() throws InterruptedException {
        lock.lock();
        try {
            while (number != 0) {
                condition.await();
            }
            number++;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    public void decrement() throws InterruptedException {
        lock.lock();
        try {
            while (number == 0) {
                condition.await();
            }
            number--;
            System.out.println(Thread.currentThread().getName() + "\t" + number);
            condition.signalAll();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

}

/**
 * 一个初始化值为零的变量，两个线程去操作，一个加1一个减1，5次
 */
public class ProdConsumer_TraditonDemo {
    public static void main(String[] args) {
        ShareData shareData = new ShareData();
        new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                try {
                    shareData.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"A").start();

        new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                try {
                    shareData.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"B").start();
    }
}
```

## 5.4 Synchronized和Lock有什么区别？

## 5.5 callable接口

# 6、线程池

## 6.1 为什么使用线程池？

线程池做的工作主要事控制线程运行的数量，处理过程中将任务放入队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数量，超出数量的线程排队等候，等其他线程执行完毕，再从队列中取出任务来执行。

他的主要特点为：线程复用；控制线程数量；管理线程。

1. 降低资源消耗。通过重复利用已创建的线程，降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗资源，还会降低系统稳定性，使用线程池可以进行统一的分配，调优和监控。

## 6.2 线程池如何使用？

## 6.3 线程几个重要参数介绍？

1. corePoolSize：线程池中的常驻核心线程数。
2. maximumPoolSize：线程池能够同时执行的最大线程数，必须大于等于1。
3. keepAliveTime：多余的线程的存活时间。
4. unit：keepAliveTime的单位。
5. workQueue：任务队列，被提交但尚未执行的任务。（阻塞队列）
6. threadFactory：表示生成线程池的工作线程的线程工厂，用于创建线程，一般为默认线程工厂即可。
7. handler：拒绝策略，表示当队列满了并且工作线程大于等于线程池的最大线程数（maximumPoolSize）时，如何来拒绝来请求的Runnable的策略。

## 6.4 线程池底层工作原理？

1.在创建了线程池后，等待提交过来的任务请求。

2.当调用execute()方法添加一个请求任务时，线程池会做如下判断：

​		2.1如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个任务；

​		2.2如果正在运行的线程数量大于或等于corePoolSize，那么将这个任务放入队列；

​		2.3如果这时候队列满了且正在运行的线程数量小于maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务；

​		2.4如果队列满了且正在运行的线程数量大于或等于maximumPoolSize，那么线程池会启动饱和拒绝策略来执行。

3.当一个线程完成任务时，它会从队列中取下一个任务来执行。

4.当一个线程无事可做超过一定时间（keepAliveTime），线程池会判断：

​		4.1如果当前线程池数大于corePoolSize，那么这个线程会被停掉。

​		4.2所以线程池的所有任务完成后它最终会收缩到corePoolSize的大小。

## 6.5 线程池的4种拒绝策略

- AbortPolicy（默认）：直接抛出RejectedExecutionException异常组织线程正常运行。
- CallerRunsPolicy：“调用者运行”一种调节机制，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退给调用者，从而降低新任务的流量。
- DiscardOldestPolicy：丢弃队列中等待最久的任务，然后把当前任务加入到队列中尝试再次提交当前任务。
- DiscardPolicy：直接丢弃任务，不予任何处理也不抛出异常。如果允许任务丢失，这是最好的一种方案。

## 6.6 实际线程池的使用

线程池不允许使用Exeutors去创建，而是通过ThreadPoolExeutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

说明：Executors返回的线程池对象的弊端如下：

1. FixedThreadPool和SingleThreadPool：允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量请求，从而导致OOM。
2. CachedThreadPool和ScheduledThreadPool：允许的创建线程数量为Integer.MAX_VALUE，可能创建大量线程，从而导致OOM。

```java
package com.ly.thread;

import java.util.concurrent.*;

public class MyThreadPoolDemo {
    public static void main(String[] args) {
        ExecutorService threadPool = new ThreadPoolExecutor(
                2,
                5,
                1L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.DiscardOldestPolicy());

        try {
            for (int i = 0; i < 20; i++) {
                threadPool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 线程正在运行！");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            threadPool.shutdown();
        }

    }

    /**
     * 不合理的使用线程池的方式
     */
    private static void threadPoolInit() {
        System.out.println(Runtime.getRuntime().availableProcessors());
        //ExecutorService threadPool = Executors.newFixedThreadPool(5);        //1池5个处理线程
        //ExecutorService threadPool = Executors.newSingleThreadExecutor();        //1池1线程
        //1池N个线程
        ExecutorService threadPool = Executors.newCachedThreadPool();

        try {
            for (int i = 0; i < 10; i++) {
                threadPool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 线程正在运行！");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            threadPool.shutdown();
        }
    }
}
```

## 6.7 合理配置线程池

> CPU密集型

- 和内存打交道，大量计算。例如大量的计算和正则匹配。
- 如何配置：CPU密集型任务应配置尽可能小的线程，如配置Ncpu+1个线程的线程池（Ncpu是处理器的核的数目）。如果线程太多，会造成线程在CPU内部的上下文切换。CPU的线程的上下文切换比指令执行耗时更多。

> IO密集型

- 解释：和磁盘，网络，文件，数据库交互很多的。
- 如何配置：由于IO密集型任务线程并不是一直在执行任务，不会经常在CPU内切换线程上下文，则应配置尽可能多的线程，如2*Ncpu。或者 核心线程数 = CPU核数 / （1-阻塞系数）   例如阻塞系数 0.8，CPU核数为4，则核心线程数为20。
- 对于IO型的任务的最佳线程数，有个公式可以计算Nthreads = Ncpu * Ucpu * （1+W/C）。
- Ncpu是处理器的核数。
- Ucpu是期房cpu的利用率（该值应介于0和1之间）。
- W/C是等待时间与计算时间的比率。等待时间与计算时间我们可以在Linux下使用相关的vmstat命令或top命令查看。

> 混合型任务

解释：前两者的混合。

如何配置：如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务。

- 只要这两个任务执行的时间相差不大，那么分解后执行的吞吐量将高于串行执行的吞吐量。
- 如果这两个任务执行的实际相差太大（例如：一个是常量时间，一个是正无穷），则没必要分解。
- 可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

# 7、死锁编码及定位分析

产生死锁的主要原因：

1. 系统资源不足
2. 进程运行推进的顺序不合适
3. 资源分配不当

死锁demo：

```java
package com.ly.thread;

import java.util.concurrent.TimeUnit;

class HoldLockThread implements Runnable {
    private String lockA;
    private String lockB;

    public HoldLockThread(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }


    @Override
    public void run() {
        synchronized (lockA) {
            System.out.println(Thread.currentThread().getName()+"持有" + lockA + "尝试获取"+lockB);
            try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
            synchronized (lockB){
                System.out.println(Thread.currentThread().getName()+"持有"+lockB+"尝试获取"+lockA);
            }
        }
    }
}

public class DeadLockDemo {
    public static void main(String[] args) {
        String lockA = "lockA";
        String lockB = "lockB";
        new Thread(new HoldLockThread(lockA,lockB),"A").start();
        new Thread(new HoldLockThread(lockB,lockA),"B").start();
    }
}
```