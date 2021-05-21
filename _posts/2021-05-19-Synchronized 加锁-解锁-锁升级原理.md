---
layout: post
title:  "Synchronized 加锁-解锁-锁升级原理"
author: "YaoFly"
tags: ["Java", "Jvm", "biased lock", "thin lock", "fat lock", "inflation"]
---   
# Synchronized 加锁-解锁-锁升级 原理

在网上看了一堆资料都只是云里雾里，只讲了大概流程，具体实现原理不够清晰。后来跟随R大([RednaxelaFX](https://www.zhihu.com/people/rednaxelafx))的指路阅读了几篇相关的原版英文论文，豁然开朗。所以写下这篇总结归纳，不然过几个礼拜学来的知识又消失在空气中了 = =！

### Mark word

首先 Java对象头里的空间分为两部分，一部分用于存储指向元空间对应Class数据的指针，另一部分用于存储hashcode、synchronized 、GC等相关信息，这个部分就叫做mark word。

![](/images/mark%20word.png)

图3.1展示了mark word 在各个阶段内存状态，其中有两个bit位固定用来标志对象当前所处无锁、偏向锁、轻量锁、重量锁等状态，接下来就主要根据这张图解析synchronized锁的相关概念。

### 偏向锁（Biased lock）

图3.1中倒数第三个bit位用于控制偏向锁的开关，如果bit位为0，则表示禁用偏向锁功能，反之开启。

图3.5所示，当请求偏向锁的时候，会通过CAS将当前线程的引用赋值到mark word中，同时Lock records锁记录列表中会有一条锁记录，锁记录预留了尚未使用的mark word存储空间和指向被锁对象的引用，此锁记录用于之后可能膨胀为轻量锁时使用。

![biased lock](/images/biased lock.png)

由于偏向锁加锁的时候把当前线程ID赋值到mark word的位置与Hashcode的存放地址重叠。所以计算过hashcode的对象无法使用偏向锁，同理正处于偏向锁的对象如果调用了hashcode计算，则会撤销偏向锁，膨胀为重量锁。

### 轻量锁（Thin lock）

轻量锁又被称作栈锁（stack locks），当一个线程请求轻量锁时，会将mark word中的信息拷贝至当前栈内存中一个叫做lock records的结构里，同时通过CAS修改锁标志位为00（轻量锁状态），并且在mark word 中保存一个指向此lock record记录的指针（displaced header reference）。线程通过判断mark word中的指针是否指向自己栈中的lock record记录来判断是否拥有锁。

![](/images/thin%20locks.png)

lock record记录包含两个部分，一个是从mark word拷贝过来的对象头信息，一个是指向被锁住对象的指针。

### 重量锁（Fat lock）

在重量级锁中没有竞争到锁的对象会 park 被挂起，退出同步块时 unpark 唤醒后续线程。唤醒操作涉及到操作系统调度会有额外的开销。持有的锁结构是一个ObjectMonitor对象，ObjectMonitor 中包含一个同步队列（由 _cxq 和 _EntryList 组成）一个等待队列（ _WaitSet ）

- 被notify或 notifyAll 唤醒时根据 policy 策略选择加入的队列（policy 默认为 0）
- 退出同步块时根据 QMode 策略来唤醒下一个线程（QMode 默认为 0）

### 锁膨胀（Inflation）

当轻量锁获取失败时（CAS fail），线程尝试将锁膨胀为重量锁。为了保证膨胀过程mark word 不会其他线程改变，首先通过CAS将mark word置NULL(0)，即图3.1中的inflating中间状态，此时持有锁的线程无法释放锁，与其他线程一同等待锁膨胀完成。膨胀循环过程中会判断以下几种锁状态做不同的处理：

- ##### 重量锁膨胀完成（锁标志位10）

  已经有其他线程将锁膨胀完毕，退出膨胀循环流程

- ##### 锁膨胀中（mark word 为Null）

  已经有其他线程正在操作锁膨胀，保持等待直到锁膨胀完成。

- ##### 锁状态为轻量锁（锁标志位00）

  分配ObjectMonitor对象，然后通过CAS将mark word置为inflating状态，如果CAS失败，则释放ObjectMonitor对象，重试膨胀循环。如果CAS成功，启用monitor对象，将mark word的值复制到ObjectMonitor中，同时把ObejctMonitor对象的引用填入mark word里。

- ##### 无锁状态（锁标志位01）

  分配ObejctMonitor对象，尝试通过CAS将monitor对象的引用赋值到mark word中，如果失败释放monitor并且重试膨胀循环，否则膨胀锁成功跳出循环。

  ![](/images/fat%20locks.png)

  重量锁膨胀过程如图3.4所示，JVM在STW期间会释放闲置的Monitor对象，从而使不再竞争的锁回到轻量锁或者偏向锁的状态。因为STW期间不会有线程再去操作锁，所以这个锁降级操作是安全的。



## 参考

- https://www.zhihu.com/question/55075763    R大指路的知乎回答
- [Evaluating and improving biased locking in the HotSpot Virtual Machine](https://link.zhihu.com/?target=http%3A//www.diva-portal.org/smash/get/diva2%3A754541/FULLTEXT01.pdf)   

