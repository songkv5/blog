---
author: willis
date: 2020-05-12 23:23
---

## ReentrantLock 原理
> ReentrantLock是JDK中J.U.C包中提供的可重入锁的java代码实现，实现基于CAS（compare and swap），能cas则得锁，否则自旋尝试再次获取，知道拿到锁。

### 源码解析
> ReentrantLock锁支持两种锁机制，一种是公平锁(FairSync实现)，一种是非公平锁（NonfairSync实现）。
FairSync与NonfairSync的相关类图如：
![ReentrantLock类图]({{ site.baseurl }}/images/2020-05-12/ReentrantLock-class.png)

>> 公平锁： 线程获得锁的顺序完全按照线程的加锁顺序来实现，严格按照线程等待队列的顺序获得锁；
>> 非公平锁：线程获得锁的顺序完全由系统控制，在尝试获得锁的时候不考虑等待队列，只有发生锁竞争的时候才会将线程放到等待队列中。
>> 公平锁月非公平锁在实现的区别在于：在锁状态控制位（state）=0的时，公平锁在尝试获得锁的时候要判断当前线程是否是线程等待队列的队首元素，如果不是，就将线程放到等待队列中；
>> 而非公平锁在尝试获得锁的时候不需要要判断当前线程是否是线程等待队列的队首元素，只要能cas成功，就能取得锁。
>> 由于ReentrantLock是可重入锁，在state>0的时候，如果能判断当前线程是锁的独占线程，也能获得锁，同时state增加1。

# 未完待续