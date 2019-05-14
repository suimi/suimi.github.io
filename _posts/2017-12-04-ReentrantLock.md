---
layout: post
title: ReentrantLock
tags: 并发编程 ReentrantLock
categories: 编程语言
---
* TOC
{:toc}

![ReentrantLock](/static/img/ReentrantLock.png)

# LockSupport
LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。
LockSupport中的park() 和 unpark() 的作用分别是阻塞线程和解除阻塞线程，而且park()和unpark()不会遇到“Thread.suspend 和 Thread.resume所可能引发的死锁”问题。
因为park() 和 unpark()有许可的存在；调用 park() 的线程和另一个试图将其 unpark() 的线程之间的竞争将保持活性。


# Condition
Condition的作用是对锁进行更精确的控制。Condition中的await()方法相当于Object的wait()方法，Condition中的signal()方法相当于Object的notify()方法，Condition中的signalAll()相当于Object的notifyAll()方法。不同的是，Object中的wait(),notify(),notifyAll()方法是和"同步锁"(synchronized关键字)捆绑使用的；而Condition是需要与"互斥锁"/"共享锁"捆绑使用的。

# ReentrantLock

ReentrantLock是一个可重入的互斥锁，又被称为“独占锁”。

顾名思义，ReentrantLock锁在同一个时间点只能被一个线程锁持有；而可重入的意思是，ReentrantLock锁，可以被单个线程多次获取。

## 过程分析

```java
// 仓库
class Depot {
    private int capacity;    // 仓库的容量

    private int size;        // 仓库的实际数量

    private Lock lock;        // 独占锁

    private Condition fullCondtion;            // 生产条件

    private Condition emptyCondtion;        // 消费条件

    public Depot(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        this.lock = new ReentrantLock();
        this.fullCondtion = lock.newCondition();
        this.emptyCondtion = lock.newCondition();
    }

    public void produce(int val) {
        //获取独占锁,如果获取不到，将会放入sync队列,设置waitStatus=SIGNAL,同时调用LockSupport.park()阻塞当前线程
        lock.lock();
        System.out.println(Thread.currentThread().getName() + " produce locked");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            // left 表示“想要生产的数量”(有可能生产量太多，需多此生产)
            int left = val;
            while (left > 0) {
                // 库存已满时，等待“消费者”消费产品。
                while (size >= capacity) {
                    System.out.println(Thread.currentThread().getName() + " produce  wait");
                    //condition等待,将放入condition队列，设置waitStatus=CONDITION, 同时调用fullyRelease,fullyRelease会释放当前线程持有的所有锁,setExclusiveOwnerThread(null),同时调用LockSupport.unpark()唤醒sync队列首线程
                    fullCondtion.await();
                    System.out.println(Thread.currentThread().getName() + " produce wait down");
                }
                // 获取“实际生产的数量”(即库存中新增的数量)
                // 如果“库存”+“想要生产的数量”>“总的容量”，则“实际增量”=“总的容量”-“当前容量”。(此时填满仓库)
                // 否则“实际增量”=“想要生产的数量”
                int inc = (size + left) > capacity ? (capacity - size) : left;
                size += inc;
                left -= inc;
                System.out.printf("%s produce(%3d) --> left=%3d, inc=%3d, size=%3d\n", Thread.currentThread().getName(),
                        val, left, inc, size);
                // 通知“消费者”可以消费了。
                //会将condition队列的firstWaiter从condition队列转移到sync队列，同时调用LockSupport.unpark()唤醒
                emptyCondtion.signal();
            }
        } catch (InterruptedException e) {
        } finally {
            System.out.println(Thread.currentThread().getName() + " produce unlock");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //释放一个锁资源,如果锁持有者为0，则会调用setExclusiveOwnerThread(null);放弃独占线程持有，同时唤醒head.next线程
            lock.unlock();
        }
    }

    public void consume(int val) {
        lock.lock();
        System.out.println(Thread.currentThread().getName() + " consume locked");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            // left 表示“客户要消费数量”(有可能消费量太大，库存不够，需多此消费)
            int left = val;
            while (left > 0) {
                // 库存为0时，等待“生产者”生产产品。
                while (size <= 0) {
                    System.out.println(Thread.currentThread().getName() + " consume  wait");
                    emptyCondtion.await();
                    System.out.println(Thread.currentThread().getName() + " consume  wait down");
                }
                // 获取“实际消费的数量”(即库存中实际减少的数量)
                // 如果“库存”<“客户要消费的数量”，则“实际消费量”=“库存”；
                // 否则，“实际消费量”=“客户要消费的数量”。
                int dec = (size < left) ? size : left;
                size -= dec;
                left -= dec;
                System.out.printf("%s consume(%3d) <-- left=%3d, dec=%3d, size=%3d\n", Thread.currentThread().getName(),
                        val, left, dec, size);
                fullCondtion.signal();
            }
        } catch (InterruptedException e) {
        } finally {
            System.out.println(Thread.currentThread().getName() + " consume unlock");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.unlock();
        }
    }

    public String toString() {
        return "capacity:" + capacity + ", actual size:" + size;
    }
}

// 生产者
class Producer {
    private Depot depot;

    public Producer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程向仓库中生产产品。
    public void produce(final int val) {
        new Thread() {
            public void run() {
                depot.produce(val);
            }
        }.start();
    }
}

// 消费者
class Customer {
    private Depot depot;

    public Customer(Depot depot) {
        this.depot = depot;
    }

    // 消费产品：新建一个线程从仓库中消费产品。
    public void consume(final int val) {
        new Thread() {
            public void run() {
                depot.consume(val);
            }
        }.start();
    }
}

public class ReentrantLockTest {

    public static void main(String[] args) {
        Depot mDepot = new Depot(100);
        Producer mPro = new Producer(mDepot);
        Customer mCus = new Customer(mDepot);

        mPro.produce(60);
        mPro.produce(120);
        mCus.consume(90);
        mCus.consume(150);
        mPro.produce(110);
        System.out.println("主线程完成");
    }
}
```

## `lock()` 和 `lockInterruptibly()` 区别
ReentrantLock的中断和非中断加锁模式的区别在于：线程尝试获取锁操作失败后，在等待过程中，如果该线程被其他线程中断了，它是如何响应中断请求的。lock方法会忽略中断请求，继续获取锁直到成功；而lockInterruptibly则直接抛出中断异常来立即响应中断，由上层调用者处理中断。


参考:
- [互斥锁ReentrantLock](http://www.cnblogs.com/skywang12345/p/3496101.html)
