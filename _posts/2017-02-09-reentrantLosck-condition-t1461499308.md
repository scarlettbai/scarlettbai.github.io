---
layout: post
title: "【并发编程】concurrent包的Condition使用详解"
description: "concurrent包的Condition使用详解."
tags: [JAVA,多线程,并发编程]
image:
  background: triangular.png
---



> 最近使用Condition时有一些疑惑，于是自己做了几个实验，了解了Condition的具体用法，现记录如下。
 
 &nbsp; &nbsp; &nbsp; &nbsp;首先我们直接来看一段之前网上看到的一个输出1-9的例子，当然网上那个例子是有缺陷的，并不能保证每次都能输出1-9，后面会介绍原因，先看如下代码：

```java
public class ConditionTest {

    public static void main(String[] args) {
        final AtomicInteger value = new AtomicInteger(1);
        final Lock lock = new ReentrantLock();
        final Condition reachThreeCondition = lock.newCondition();
        final Condition reachSixCondition = lock.newCondition();

        Thread threadA = new Thread(new Runnable() {
            public void run() {
                lock.lock();
                System.out.println("线程1拿到锁【1】");
                try {
                    // 输出1-3 之后激活reachThreeCondition
                    while (value.get() <= 3) {
                        System.out.println(value.get());
                        value.addAndGet(1);
                    }
                    reachThreeCondition.signal();
                    System.out.println("线程1【1】调用唤醒【reachThreeCondition】");
                } finally {
                    lock.unlock();
                    System.out.println("线程1【1】释放锁");
                }
                lock.lock();
                System.out.println("线程1拿到锁【2】");
                try {
                    // 挂起等待唤醒reachSixCondition操作则继续输出后续数字
                    System.out.println("线程1【2】进入等待");
                    reachSixCondition.await();
                    System.out.println("线程1【2】被唤醒");
                    // 输出剩余数字
                    while (value.get() <= 9) {
                        System.out.println(value.get());
                        value.addAndGet(1);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }

        });

        Thread threadB = new Thread(new Runnable() {
            public void run() {
                try {
                    lock.lock();
                    System.out.println("线程2拿到锁【1】");
                    while (value.get() <= 3) {
                        // 线程挂起等待输出1-3之后唤醒
                        System.out.println("线程2【1】进入等待");
                        reachThreeCondition.await();
                        System.out.println("线程2【1】被唤醒");
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                    System.out.println("线程2【1】释放了锁");
                }
                try {
                    lock.lock();
                    System.out.println("线程2拿到锁【2】");
                    while (value.get() <= 6) {
                        System.out.println(value.get());
                        value.addAndGet(1);
                    }
                    // 唤醒reachSixCondition继续输出后续数字
                    reachSixCondition.signal();
                    System.out.println("线程2【2】唤醒了【reachSixCondition】");
                } finally {
                    lock.unlock();
                    System.out.println("线程2【2】释放了锁");
                }
            }

        });

        // 启动两个线程
        threadB.start();
        threadA.start();
    }
}
```

输出结果1：

```java
线程2拿到锁【1】
线程2【1】进入等待
线程1拿到锁【1】
1
2
3
线程1【1】调用唤醒【reachThreeCondition】
线程1【1】释放锁
线程1拿到锁【2】
线程1【2】进入等待
线程2【1】被唤醒
线程2【1】释放了锁
线程2拿到锁【2】
4
5
6
线程2【2】唤醒了【reachSixCondition】
线程2【2】释放了锁
线程1【2】被唤醒
7
8
9
```
输出结果2：

```java
线程2拿到锁【1】
线程2【1】进入等待
线程1拿到锁【1】
1
2
3
线程1【1】调用唤醒【reachThreeCondition】
线程1【1】释放锁
线程2【1】被唤醒
线程2【1】释放了锁
线程2拿到锁【2】
4
5
6
线程2【2】唤醒了【reachSixCondition】
线程2【2】释放了锁
线程1拿到锁【2】
线程1【2】进入等待
```

输出结果3：
```java
线程1拿到锁【1】
1
2
3
线程1【1】调用唤醒【reachThreeCondition】
线程1【1】释放锁
线程1拿到锁【2】
线程1【2】进入等待
线程2拿到锁【1】
线程2【1】释放了锁
线程2拿到锁【2】
4
5
6
线程2【2】唤醒了【reachSixCondition】
线程2【2】释放了锁
线程1【2】被唤醒
7
8
9
```

 &nbsp; &nbsp; &nbsp; &nbsp;上面列出了三种输出结果，结果1其实是一个很理想的运行结果，所有代码都执行完了，其实读者可以反复运行上面代码，会发现可能出现好几种结果，这里只列出三种典型的结果。
 &nbsp; &nbsp; &nbsp; &nbsp;我们来根据结果做如下分析：
 1. 线程1和线程2同时启动，此时**首次出现锁竞争**，上述结果1是线程2获取到锁（此处也可能是线程1获取到锁，见结果3），则线程1处于挂起状态，等待获取锁，线程2获取锁后判断满足条件，则**使用reachThreeCondition.await()让线程进入挂起状态，且此处会释放锁**。
 2. 由于上一步**线程2释放了锁，所以此时线程1可获取到锁**，于是线程1获取到锁，并继续执行输出123，之后调用reachThreeCondition.signal()唤醒线程2，此时线程1和线程2同时运行，线程1和2分别继续执行后续释放锁的代码（此时线程2其实没有锁可释放）。
 3. **此时又会出现锁竞争，线程1和线程2会竞争锁**（结果1和结果3是由线程1拿到了锁，结果2是由线程2拿到了锁），此时分两种情况：
 a.（1）线程1拿到锁，之后满足条件通过reachSixCondition挂起并释放锁，（2）此时线程2拿到锁输出456之后唤醒reachSixCondition，之后俩线程继续各自执行后续代码到程序结束，完整输出1-9.
 b.线程2拿到锁，之后不满足条件释放锁，此时**第三次出现锁竞争**,会出现两种结果，1）线程1拿到锁则执行a逻辑，完整输出1-9；2）线程2拿到锁，则执行线程2输出4-6，之后释放锁，线程1获取锁，之后执行reachSixCondition挂起，由于此时线程2已执行结束，则线程1的reachSixCondition没有地方被唤醒，所以线程1会一直处于挂起状态。

所以通过上面跟踪代码，理解了Condition的使用，**其实在于可以根据不同条件来挂起、唤醒线程，且会自动释放锁，以达到可以更加方便的实现线程的有序性**。

----------
欢迎关注个人微信公众号：<br/>
![](/images/weixin.jpg)