---
title: AbstractQueuedSynchronizer 源码分析（共享锁）
date: 2017-12-21 18:27:44
layout: tag
tags: [思考]
---


# 源码看之前的问题
- race condition如何避免？
- 工作流程是怎么样的？
- 使用什么方式实现的？

<!--more-->

# 使用到的其他类说明和资料
## LockSupport 简要说明
在AbstractQueuedSynchronizer中使用LockSupport类来实现线程的挂起和唤醒，对应方法分别我park和unpark，内部实现原理是代理给了unsafe包的park和unpark

### 为何使用park和unpark
Thread中提供了suspend和resume两个方法，不过这两个方法有很重大的缺陷，就是在suspend之前调用了resume，resume操作时没有任何作用的，线程会一直挂起再也得不到运行，目前这两个方法已经不建议使用。

park会阻塞线程直到unpark调用，但unpark操作不依赖于park，在调用park之前调用了unpark对线程一样有效(park之前检查unpark状态应该是)，而且多次调用unpark只对后面的一次park起作用。由于前面遗留的unpark操作影响，调用park后可能会立即返回。不过下一次park又会继续阻塞等待unpark。

其次park还支持超时，获取锁时的超时策略就依赖于它。


## Unsafe类相关说明
在多线程环境下对一个值进行操作时需要保证原子性，lock类使用了Unsafe类中的compareAndSet等CAS方法来保证操作的原子性，在不成功的情况下会自旋重试
Unsafe类是sun.misc包下的类，由于其安全策略，应用程序中写的类是无法使用这个类的，而且其中实现大部分都是native的，了解一下API功能，不影响阅读jdk源码就可以了

## Doug Lea大神的paper
地址：http://gee.cs.oswego.edu/dl/papers/aqs.pdf
详细讲述了aqs的设计过程，上面的park与unpark就翻译自里面的一段。


# AbstractQueuedSynchronizer 源码解析
ps：condition相关的先不涉及，单纯的看lock相关源码
ps2:单独看AQS很抽象，我们结合具体类来了解相关功能
ps3:要用多线程的思维去看，单线程思维看这个根本就看不明白

## 重要的属性字段
1. state
  标识当前锁的状态，源码实现中一般标识锁数量，像在CountDownLatch中state标识latch的count，每当有线程countDown时，state就减一，ReentrantLock标识锁的重入次数，进入+1，释放-1
2. head,tail
  队列的头尾，下面会说明下队列

## 内部类Node
AbstractQueuedSynchronizer维护了一个FIFO的队列，每个队列节点就是一个Node，Node中维护了前后节点(pre,next)的信息，和每个节点的waitStatus以及节点的模式(共享还是独占)，在获取锁失败后就会加入到队列末尾，拥有锁的线程释放锁后会通知队列中的第一个节点。

waitStatus有几个状态和约定
| 值 | 说明 |
|:---:|:---:|
| >0 | 无效状态，说明node不再竞争锁 |
| <0 | 有效状态，node正在竞争锁 |
| 1  | CANCELLED，被取消 |
| 0  | 初始化状态，表示SYNC |
| -1 | SIGNAL 表示后继节点需要被唤醒 |
| -2 | CONDITION 表示线程在等待condition|
| -3 | PROPAGATE 表示下一次acquireShared应该被无条件传播|

mode：
| 值 | 说明 |
|:---:|:---:|
|SHARED | 共享模式 |
|EXCLUSIVE| 独占模式|

## CountDownLatch 对AQS的使用
我们从最简单的CountDownLatch来看一下AQS的共享模式的使用

### demo以及CountDownLatch相关API
jdk中的demo
```java
class Driver2 { // ...
  void main() throws InterruptedException {
    CountDownLatch doneSignal = new CountDownLatch(N);
    Executor e = ...

    for (int i = 0; i < N; ++i) // create and start threads
      e.execute(new WorkerRunnable(doneSignal, i));

    doneSignal.await();           // wait for all to finish
  }
}

class WorkerRunnable implements Runnable {
  private final CountDownLatch doneSignal;
  private final int i;
  WorkerRunnable(CountDownLatch doneSignal, int i) {
    this.doneSignal = doneSignal;
    this.i = i;
  }
  public void run() {
    try {
      doWork(i);
      doneSignal.countDown();
    } catch (InterruptedException ex) {} // return;
  }

  void doWork() { ... }
}
```
main函数生成了N个任务放到线程池中异步执行，每个任务执行完毕后会countdown一下表明任务完成，主线程一直await到所有的任务执行完毕才会退出

内部类Sync继承了AQS，重载了share相关的两个方法
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }
    /**
      * 返回值>=0 表示获取锁成功，
      * >0 表示需要向后传播 =0不向后传播
      * <0 表示获取锁失败
     */
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```
1.tryAcquireShared 当前状态等于0，获取成功，即所有线程准备完毕
2.tryReleaseShared 释放锁时将state减一，里面用到了CAS来保证操作的原子性

CountDownLatch相关方法：
```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public void countDown() {
    sync.releaseShared(1);
}
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```
全部都代理给了Sync类

### countDown使用的releaseShared方法
countDown使用的releaseShared方法比较简单，先来看一下
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {    //1
        doReleaseShared();
        return true;
    }
    return false;
}
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))   //2
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))   //3
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed   //4
            break;
    }
}
```
1.尝试释放共享锁，上面介绍了，就是state-1，成功后执行后续操作
2.获取队列head，当waitStatus为SIGNAL，就将其设置为0，设置成功后唤醒后继节点，不成功继续自旋尝试
3.head状态为0，将自身状态设置为propagate，这里ws为0，在后面可以看到其实是因为没有后续节点
4.如果在此过程中head改变了，就再次循环检查。后面我们会看到在线程获取到了锁之后，也还会调用这个方法来通知后继节点，这样前驱通知后继，扩散到了整个队列中，使所有节点都接收到了唤醒通知


### await使用的acquireSharedInterruptibly方法
再来看一下await中的acquireSharedInterruptibly实现
```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)               // 1
        doAcquireSharedInterruptibly(arg);
}
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);     // 2
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();  // 3
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);   
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&   // 4
                parkAndCheckInterrupt())                   //5
                throw new InterruptedException();
        }
    } finally {
        if (failed)                          // 6
            cancelAcquire(node);
    }
}
```
1.尝试获取锁，获取成功直接返回，获取不成功进入doAcquireSharedInterruptibly
2.将当前线程封装成node加入到队列中
3.获得node的前驱节点，如果前驱节点为head节点，那么再次尝试获取锁，获取成功后将node设置为head节点，并向后传播
4.在获取失败后检查状态是否需要挂起，如果是，就挂起并在唤醒后检查中断状态(唤醒后线程是从挂起的位置继续往下执行)
5.失败将当前node置为取消，失败从代码看只有一种情况，就是被中断后抛出异常

分步骤说明，不按上述顺序，见标号：
2.加入到队列中addWaiter方法
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
先自己尝试一下加入队列，如果失败就进入enq方法入队，可以看到，队列初始化时放置了一个空节点作为头部，线程封装的node加入到了其后

4.然后我们先看一下shouldParkAfterFailedAcquire，因为这个节点会改变waitStatus，对后面的propagate会有影响
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)      //1
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {              //2
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {                    
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);     
        pred.next = node;
    } else {                   //3
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
分三步
1.前驱节点waitStatus为SIGNAL直接返回true，表示可以挂起
2.waitStatus大于0表示前驱节点已经被取消或其他无效状态，将其清理出队列，然后返回false，doAcquireSharedInterruptibly会自旋一次
3.这个else里waitStatus要么是初始化时的0，要么就是被其他线程设置成了propagate，将waitStatus设置为SIGNAL，然后返回false，doAcquireSharedInterruptibly会自旋一次

可以看到每当有一个新线程进入等待队列时，都会把前一个节点的waitStatus变为SIGNAL,表示后继节点需要被通知唤醒，新入队的节点waitStatus为SYNC
> head
> head(-1)->node1(0)
> head(-1)->node1(-1)->node2(0)



3.将获取到锁的节点设置为head，并向后传播setHeadAndPropagate
```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
//头结点对应的线程已经获得了锁，
//相当于于出队，这个节点已经不再竞争锁了
//再竞争锁会再加入到队列中
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```
propagate(tryAcquireShared返回值) > 0 表示需要向后传播
h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0 或头结点为空或状态为有效

通知后继节点doReleaseShared上面已经说过了。


我们分几种情况讨论一下
> 1. await直接获取到锁，也就是所有任务已经完成，那么直接返回，继续执行
> 2. 任务没有完成，await获取锁失败，进入FIFO队列等待
> > 2.1 任务完成后，调用doReleaseShared通知后继节点，将队列中的第一个node设置为head，并再次调用doReleaseShared
> > 2.2 一直到队列末尾，所有节点获取到锁，通知完毕，所有线程获取到共享锁，继续执行