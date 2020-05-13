---
author: willis
date: 2020-05-12 23:23
---

## ReentrantLock 原理
> ReentrantLock是JDK中J.U.C包中提供的可重入锁的java代码实现，实现基于CAS（compare and swap），能cas则得锁，否则自旋尝试再次获取，知道拿到锁。

### 源码解析
> ReentrantLock锁支持两种锁机制，一种是公平锁(FairSync实现)，一种是非公平锁（NonfairSync实现）。
> FairSync与NonfairSync的相关类图如：

![ReentrantLock类图]({{ site.baseurl }}/images/2020-05-12/ReentrantLock-class.png)

> 公平锁： 线程获得锁的顺序完全按照线程的加锁顺序来实现，严格按照线程等待队列的顺序获得锁；
> 非公平锁：线程获得锁的顺序完全由系统控制，在尝试获得锁的时候不考虑等待队列，只有发生锁竞争的时候才会将线程放到等待队列中。
> 公平锁月非公平锁在实现的区别在于：在锁状态控制位（state）=0的时，公平锁在尝试获得锁的时候要判断当前线程是否是线程等待队列的队首元素，如果不是，就将线程放到等待队列中；
> 而非公平锁在尝试获得锁的时候不需要要判断当前线程是否是线程等待队列的队首元素，只要能cas成功，就能取得锁。
> 由于ReentrantLock是可重入锁，在state>0的时候，如果能判断当前线程是锁的独占线程，也能获得锁，要对state增加1。

#### 1. NonfairSync
```java
/**
     * Sync object for non-fair locks
     */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     * 拿锁操作
     */
    final void lock() {
    	// 直接通过cas进行锁状态判断，如果state==0，代表锁是可以被获取到，cas就会成功，将当前线程设置为锁的独占线程即可
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else// 如果获取不到锁，则进行尝试获得锁的操作，参考父类AbstractQueuedSynchronizer
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```
acquire方法取自父类AbstractQueuedSynchronizer，代码如下
```java
public final void acquire(int arg) {
    //如果尝试获得锁还是失败(tryAcquire返回false)，那么就会将线程以独占锁的方式（EXCLUSIVE）放入等待队列（链表实现，参考Node类）。
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
其中nonfairTryAcquire方法是父类Sync中的方法，如下
```java
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
> 以上是非公平锁的主要相关代码。java.util.concurrent.locks.ReentrantLock.NonfairSync.lock()的操作可知, 非公平锁首先通过cas进行锁状态判断，如果state==0，代表锁是可以被竞争获得的状态，cas就会成功，将当前线程设置为锁的独占线程即可；如果获取不到锁，则进行尝试获得锁的操作，acquire方法进入tryAcquire方法，进而转到nonfairTryAcquire方法中，看方法实现，会尝试通过cas获得锁，成功的话则获得锁，如果是重入，则state+1，否则获得锁失败（返回false）。

> 在重入锁实现中，state只能同时被一个线程修改，通过lock的+1后，用户线程自己调用unlock释放锁后会-1。所以，一旦锁正常释放，state一定变回0的

> 再回到acquire方法，如果上一步尝试获得锁还是失败，则通过acquireQueued(addWaiter(Node.EXCLUSIVE), arg)操作将当前线程以独占锁的方式（EXCLUSIVE）放入等待队列（链表实现，参考Node类）。Node的类定义如下：

```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     * 用于标记锁的类型，如果是EXCLUSIVE代表是独占锁，如果是SHARE代表是共享锁
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     *
     * @return the predecessor of this node
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

#### 2. FairSync
```java
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        	// hasQueuedPredecessors,这个方法的结果，如果当前线程是不是队首结点，则返回真。!hasQueuedPredecessors()的断言表示如果当前线程是队首结点，也就是站在排队的最前面
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```
hasQueuedPredecessors的源码如下
```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
    	//head不关联线程，head的next是队列首结点。所以s.thread != Thread.currentThread()表示，队首结点关联的线程不是当前线程
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
**这里需要提醒一点，在等待队列的结构中，head并不关联线程，head->next是指向的等待队列的第一个结点，关联第一个线程**

> 反观公平锁的实现，与非公平锁的不同之处有如下几点：
>>1. lock方法中，不会让线程直接做cas操作，没有这一步判断：if (compareAndSetState(0, 1)), 直接调用尝试获得锁的方法，也就是说，公平锁不能根据当前状态是不是0来判定，你这个线程能不能拿到所。
>>2. acquire方法与非公平锁一样的实现，主要区别在tryAcquire方法。公平锁的tryAcquire方法重点就在，判断state==0的时候，要判断当前线程是不是等待队列的队首结点，如果是队首结点，才能获得锁。
>因此在公平锁中，想获得锁就必须得先排队，谁站在队前面谁先得锁。

#### 3. 等待对立中的线程怎么办？
回观acquire方法，其实公平锁和非公平锁都有调用，在lock方法体中，源码再看一遍
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
其中，acquireQueued方法就是为了处理等带队列中的线程，源码：
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
        	// p是node的前驱结点
            final Node p = node.predecessor();
            // 判断p==head，就是判断如果node是队首元素。也就是node必须是队首元素，才会拿到锁。换句话，也就是，等待队列中的线程，就是按照入队的顺序来获得所，不管你是公平锁还是非公平锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // shouldParkAfterFailedAcquire方法负责把取消的线程结点移除
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
> 所以，无论是公平锁还是非公平锁，一旦进入了等待队列，都是按照先进先出的顺序依次进行获得锁的。