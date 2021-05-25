---
layout: post
title:  "Synchronized 加锁-解锁-锁升级原理"
author: "YaoFly"
tags: ["Java", "Jvm", "biased lock", "thin lock", "fat lock", "inflation"]
---   
在网上看了一堆资料都只是云里雾里，只讲了大概流程，具体实现原理不够清晰。后来跟随R大([RednaxelaFX](https://www.zhihu.com/people/rednaxelafx))的指路阅读了几篇相关的原版英文论文，豁然开朗。所以写下这篇总结归纳，不然过几个礼拜学来的知识又消失在空气中了 = =！

### Mark word

首先 Java对象头里的空间分为两部分，一部分用于存储指向元空间对应Class数据的指针，另一部分用于存储hashcode、synchronized 、GC等相关信息，这个部分就叫做mark word。

![](/images/mark word.png)

图3.1展示了mark word 在各个阶段内存状态，其中有两个bit位固定用来标志对象当前所处无锁、偏向锁、轻量锁、重量锁等状态，接下来就主要根据这张图解析synchronized锁的相关概念。

### 轻量锁（Thin lock）

轻量锁又被称作栈锁（stack locks），当一个线程请求轻量锁时，会将mark word中的信息拷贝至当前栈内存中一个叫做lock records的结构里，同时通过CAS修改锁标志位为00（轻量锁状态），并且在mark word 中保存一个指向此lock record记录的指针（displaced header reference）。线程通过判断mark word中的指针是否指向自己栈中的lock record记录来判断是否拥有锁。

![](/images/thin locks.png)

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

  ![](/images/fat locks.png)

  重量锁膨胀过程如图3.4所示，JVM在STW期间会释放闲置的Monitor对象，从而使不再竞争的锁回到轻量锁或者偏向锁的状态。因为STW期间不会有线程再去操作锁，所以这个锁降级操作是安全的。

### 偏向锁（Biased lock）

图3.1中无锁（unlocked）和偏向锁（biased）的情况中，倒数第三个bit位用于控制偏向锁的开关，如果bit位为0，则表示禁用偏向锁功能，反之开启，在存放在元空间对象对应的class中，也存有这样一个偏向锁开关bit位，用于控制全局对象是否能启用偏向锁功能。另外JVM启动的前4s由于性能低下，期间创建的对象也会默认关闭偏向锁，4s之后的对象正常开启偏向锁功能。

##### 全局安全点（Safe point）

safe point这个词我们在GC中经常会提到，简单来说就是其代表了一个状态，在该状态下所有线程都是暂停的。对于偏向锁（Biased Lock），想要解除偏置，需要线程状态还有获取锁的线程的精确信息，所以需要等待所有线程进入全局安全点，也就是STW。

##### 批量重偏向/撤销（bulk rebias/bulk revoke）

当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时将偏向锁撤销为无锁状态或升级为轻量级/重量级锁。这个过程是要消耗一定的成本的，所以如果说运行时的场景本身存在多线程竞争的，那偏向锁的存在不仅不能提高性能，而且会导致性能下降。因此，JVM中增加了一种批量重偏向/撤销的机制。

以class为单位，为每个class维护一个偏向锁撤销计数器。每一次该class的对象发生偏向撤销操作是，该计数器+1，当这个值达到重偏向阈值(默认20)时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。当这个值达到批量撤销阈值(默认40)时，则会执行批量撤销，将class里的偏向锁开关bit位置为0，关闭该class对象的偏向锁功能。

###### 批量重偏向原理

1. 首先引入一个概念epoch，其本质是一个时间戳，代表了偏向锁的有效性，epoch存储在可偏向对象的MarkWord中。除了对象中的epoch,对象所属的类class信息中，也会保存一个epoch值。
2. 每当遇到一个全局安全点时(这里的意思是说批量重偏向没有完全替代了全局安全点，全局安全点是一直存在的)，比如要对class C 进行批量再偏向，则首先对 class C中保存的epoch进行增加操作，得到一个新的epoch_new
3. 然后扫描所有持有 class C 实例的线程栈，根据线程栈的信息判断出该线程是否锁定了该对象，仅将epoch_new的值赋给被锁定的对象中，也就是现在偏向锁还在被使用的对象才会被赋值epoch_new。
4. 退出安全点后，当有线程需要尝试获取偏向锁时，直接检查 class C 中存储的 epoch 值是否与目标对象中存储的 epoch 值相等， 如果不相等，则说明该对象的偏向锁已经无效了（因为（3）步骤里面已经说了只有偏向锁还在被使用的对象才会有epoch_new，这里不相等的原因是class C里面的epoch值是epoch_new,而当前对象的epoch里面的值还是epoch），此时竞争线程可以尝试对此对象重新进行偏向操作。

##### 请求偏向锁

请求一个偏向锁的时候，会执行一下几个步骤：

- 判断对象的偏向锁开关bit位

  如果是0，则表示无法偏向，直接进入轻量锁请求流程。

- 判断存于元空间对象的class的bit位

  如果位于class的偏向锁开关bit位为0时，则不允许偏向并且将对象的偏向锁开关bit位设为0，使用轻量锁代替。

- 判断epoch

  检查对象的epoch与class中的epoch作对比，如果不相等，则代表当前偏向锁已过期，同时允许重偏向，直接CAS修改mark word指向当前线程。

- 校验是否拥有锁

  判断mark word中的线程id是否与当前id相同，如果不匹配，假设当前为无锁可偏向状态（anonymously biased），通过CAS尝试获得锁，如果CAS失败，说明锁竞争，释放偏向锁（可能涉及到全局安全点），然后膨胀为轻量锁。

图3.5所示，当请求偏向锁的时候，会通过CAS将当前线程的引用赋值到mark word中，同时Lock records锁记录列表中会有一条锁记录，锁记录预留了尚未使用的mark word存储空间和指向被锁对象的引用，此锁记录用于记录锁重入的次数（多少条Lock records就代表偏向锁重入了多少次），同时为之后可能膨胀为轻量锁时使用。JVM会在Safe point时，直接对产生锁竞争的偏向锁进行修改，让其看起来像是一开始就使用了轻量锁一样。

![biased lock](/images/biased lock.png)

由于偏向锁加锁的时候把当前线程ID赋值到mark word的位置与Hashcode的存放地址重叠。所以计算过hashcode的对象无法使用偏向锁，同理正处于偏向锁的对象如果调用了hashcode计算，则会撤销偏向锁，膨胀为重量锁。

mark word各个状态和其之间的转换如图3.6所示：

![state graph of the mark word](/images/state graph of the mark word.png)

### 偏向锁的问题

在高并发应用中，偏向锁并不能带来性能提升，反而因为偏向锁取消带来了很多没必要的某些线程进入Safepoint 或者 Stop the world。所以建议关闭：`-XX:-UseBiasedLocking`

同时在Java 15之后，官方已经决定废弃偏向锁的设计了， 具体原因可以自行查找 [JEPS374](https://openjdk.java.net/jeps/374)。

## 参考

- https://www.zhihu.com/question/55075763    R大指路的知乎回答
- [Evaluating and improving biased locking in the HotSpot Virtual Machine](https://link.zhihu.com/?target=http%3A//www.diva-portal.org/smash/get/diva2%3A754541/FULLTEXT01.pdf)   

