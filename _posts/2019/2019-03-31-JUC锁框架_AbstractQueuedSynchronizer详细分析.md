---
layout: post
title: JUC锁框架_AbstractQueuedSynchronizer详细分析
category: Java
tags: [并发]
keywords: 并发
---

#JUC锁框架_AbstractQueuedSynchronizer详细分析/n                  
    

JUC锁框架_AbstractQueuedSynchronizer详细分析
=====================================

转 ： [wo883721](https://www.jianshu.com/p/0da2939391cf)


AQS是JUC锁框架中最重要的类，通过它来实现独占锁和共享锁的。本章是对AbstractQueuedSynchronizer源码的完全解析，分为四个部分介绍：

> 1.  CLH队列即同步队列：储存着所有等待锁的线程
> 2.  独占锁
> 3.  共享锁
> 4.  Condition条件

> 注: 还有一个AbstractQueuedLongSynchronizer类，它与AQS功能和实现几乎一样，唯一不同的是AQLS中代表锁被获取次数的成员变量state类型是long长整类型，而AQS中该成员变量是int类型。

一. CLH队列(线程同步队列)
----------------

因为获取锁是有条件的，没有获取锁的线程就要阻塞等待，那么就要存储这些等待的线程。

> 在AQS中我们使用CLH队列储存这些等待的线程，但它并不是直接储存线程，而是储存拥有线程的node节点。所以先介绍重要内部类Node。

### 1.1 内部类Node

       static final class Node {
            // 共享模式的标记
            static final Node SHARED = new Node();
            // 独占模式的标记
            static final Node EXCLUSIVE = null;
    
            // waitStatus变量的值，标志着线程被取消
            static final int CANCELLED =  1;
            // waitStatus变量的值，标志着后继线程(即队列中此节点之后的节点)需要被阻塞.(用于独占锁)
            static final int SIGNAL    = -1;
            // waitStatus变量的值，标志着线程在Condition条件上等待阻塞.(用于Condition的await等待)
            static final int CONDITION = -2;
            // waitStatus变量的值，标志着下一个acquireShared方法线程应该被允许。(用于共享锁)
            static final int PROPAGATE = -3;
    
            // 标记着当前节点的状态，默认状态是0, 小于0的状态都是有特殊作用，大于0的状态表示已取消
            volatile int waitStatus;
    
            // prev和next实现一个双向链表
            volatile Node prev;
            volatile Node next;
    
            // 该节点拥有的线程
            volatile Thread thread;
    
            // 可能有两种作用：1. 表示下一个在Condition条件上等待的节点
            // 2. 表示是共享模式或者独占模式，注意第一种情况节点一定是共享模式
            Node nextWaiter;
    
            // 是不是共享模式
            final boolean isShared() {
                return nextWaiter == SHARED;
            }
    
            // 返回前一个节点prev，如果为null，则抛出NullPointerException异常
            final Node predecessor() throws NullPointerException {
                Node p = prev;
                if (p == null)
                    throw new NullPointerException();
                else
                    return p;
            }
    
            // 用于创建链表头head，或者共享模式SHARED
            Node() {
            }
    
            // 使用在addWaiter方法中
            Node(Thread thread, Node mode) {
                this.nextWaiter = mode;
                this.thread = thread;
            }
    
            // 使用在Condition条件中
            Node(Thread thread, int waitStatus) {
                this.waitStatus = waitStatus;
                this.thread = thread;
            }
        }
    

重要的成员属性：

> 1.  waitStatus: 表示当前节点的状态，默认状态是0。总共有五个值CANCELLED、SIGNAL、CONDITION、PROPAGATE以及0。
> 2.  prev和next：记录着当前节点前一个节点和后一个节点的引用。
> 3.  thread：当前节点拥有的线程。当拥有锁的线程释放锁的时候，可能会调用LockSupport.unpark(thread)，唤醒这个被阻塞的线程。
> 4.  nextWaiter：如果是SHARED，表示当前节点是共享模式，如果是null，当前节点是独占模式，如果是其他值，当前节点也是独占模式，不过这个值也是Condition队列的下一个节点。

> 注意：通过Node我们可以实现两个队列，一是通过prev和next实现CLH队列(线程同步队列,双向队列)，二是nextWaiter实现Condition条件上的等待线程队列(单向队列)，后一个我们在Condition中介绍。

### 1.2 操作CLH队列

#### 1.2.1 存储CLH队列

        // CLH队列头
        private transient volatile Node head;
    
        // CLH队列尾
        private transient volatile Node tail;
    

#### 1.2.2 设置CLH队列头head

        /**
         * 通过CAS函数设置head值，仅仅在enq方法中调用
         */
        private final boolean compareAndSetHead(Node update) {
            return unsafe.compareAndSwapObject(this, headOffset, null, update);
        }
    

这个方法只在enq方法中调用，通过CAS函数设置head值，保证多线程安全

       // 重新设置队列头head，它只在acquire系列的方法中调用
        private void setHead(Node node) {
            head = node;
            // 线程也没有意义了，因为该线程已经获取到锁了
            node.thread = null;
            // 前一个节点已经没有意义了
            node.prev = null;
        }
    

这个方法只在acquire系列的方法中调用，重新设置head，表示移除一些等待线程节点。

### 1.2.3 设置CLH队列尾tail

        /**
         * 通过CAS函数设置tail值，仅仅在enq方法中调用
         */
        private final boolean compareAndSetTail(Node expect, Node update) {
            return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
        }
    

这个方法只在enq方法中调用，通过CAS函数设置tail值，保证多线程安全

#### 1.2.4 将一个节点插入到CLH队列尾

       // 向队列尾插入新节点，如果队列没有初始化，就先初始化。返回原先的队列尾节点
        private Node enq(final Node node) {
            for (;;) {
                Node t = tail;
                // t为null，表示队列为空，先初始化队列
                if (t == null) {
                    // 采用CAS函数即原子操作方式，设置队列头head值。
                    // 如果成功，再将head值赋值给链表尾tail。如果失败，表示head值已经被其他线程，那么就进入循环下一次
                    if (compareAndSetHead(new Node()))
                        tail = head;
                } else {
                    // 新添加的node节点的前一个节点prev指向原来的队列尾tail
                    node.prev = t;
                    // 采用CAS函数即原子操作方式，设置新队列尾tail值。
                    if (compareAndSetTail(t, node)) {
                        // 设置老的队列尾tail的下一个节点next指向新添加的节点node
                        t.next = node;
                        return t;
                    }
                }
            }
        }
    

这个方法向CLH队列尾插入一个新节点，如果队列为空，就先创建队列再插入新节点，返回老的队列尾节点。

#### 1.2.5 将当前线程添加到CLH队列尾

       // 通过给定的模式mode(独占或者共享)为当前线程创建新节点，并插入队列中
        private Node addWaiter(Node mode) {
            // 为当前线程创建新的节点
            Node node = new Node(Thread.currentThread(), mode);
            Node pred = tail;
            // 如果队列已经创建，就将新节点插入队列尾。
            if (pred != null) {
                node.prev = pred;
                if (compareAndSetTail(pred, node)) {
                    pred.next = node;
                    return node;
                }
            }
            // 如果队列没有创建，通过enq方法创建队列，并插入新的节点。
            enq(node);
            return node;
        }
    

为当前线程创建一个新节点，再插入到CLH队列尾，返回新创建的节点。

二. 独占锁
------

我们先想一想独占锁的功能是什么?

> 独占锁至少有两个功能：
> 
> 1.  获取锁的功能。 当多个线程一起获取锁的时候，只有一个线程能获取到锁，其他线程必须在当前位置阻塞等待。
> 2.  释放锁的功能。获取锁的线程释放锁资源，而且还必须能唤醒正在等待锁资源的一个线程。  
>     带着这些疑惑，我们来看AQS中是怎么实现的。

### 2.1 获取独占锁的方法

#### 2.1.1 acquire方法

       /**
         * 获取独占锁。如果没有获取到，线程就会阻塞等待，直到获取锁。不会响应中断异常
         * @param arg
         */
        public final void acquire(int arg) {
            // 1. 先调用tryAcquire方法，尝试获取独占锁，返回true，表示获取到锁，不需要执行acquireQueued方法。
            // 2. 调用acquireQueued方法，先调用addWaiter方法为当前线程创建一个节点node，并插入队列中，
            // 然后调用acquireQueued方法去获取锁，如果不成功，就会让当前线程阻塞，当锁释放时才会被唤醒。
            // acquireQueued方法返回值表示在线程等待过程中，是否有另一个线程调用该线程的interrupt方法，发起中断。
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }
    

将这个方法一下格式，大家就很好理解了

       public final void acquire(int arg) {
            // 1.先调用tryAcquire方法，尝试获取独占锁，返回true则直接返回
            if (tryAcquire(arg)) return;
            // 2. 调用addWaiter方法为当前线程创建一个节点node，并插入队列中
            Node node = addWaiter(Node.EXCLUSIVE);
            // 调用acquireQueued方法去获取锁，
            // acquireQueued方法返回值表示在线程等待过程中，是否有另一个线程调用该线程的interrupt方法，发起中断。
            boolean interrupted = acquireQueued(node, arg);
            // 如果interrupted为true，则当前线程要发起中断请求
            if (interrupted) {
                selfInterrupt();
            }
        }
    

#### 2.1.2 tryAcquire方法

        // 尝试去获取独占锁，立即返回。如果返回true表示获取锁成功。
        protected boolean tryAcquire(int arg) {
            throw new UnsupportedOperationException();
        }
    

如果子类想实现独占锁，则必须重写这个方法，否则抛出异常。这个方法的作用是当前线程尝试获取锁，如果获取到锁，就会返回true，并更改锁资源。没有获取到锁返回false。

> 注：这个方法是立即返回的，不会阻塞当前线程

下面是ReentrantLock中FairSync的tryAcquire方法实现

           // 尝试获取锁，与非公平锁最大的不同就是调用hasQueuedPredecessors()方法
            // hasQueuedPredecessors方法返回true，表示等待线程队列中有一个线程在当前线程之前，
            // 根据公平锁的规则，当前线程不能获取锁。
            protected final boolean tryAcquire(int acquires) {
                final Thread current = Thread.currentThread();
                // 获取锁的记录状态
                int c = getState();
                // 如果c==0表示当前锁是空闲的
                if (c == 0) {
                    if (!hasQueuedPredecessors() &&
                        compareAndSetState(0, acquires)) {
                        setExclusiveOwnerThread(current);
                        return true;
                    }
                }
                // 判断当前线程是不是独占锁的线程
                else if (current == getExclusiveOwnerThread()) {
                    int nextc = c + acquires;
                    if (nextc < 0)
                        throw new Error("Maximum lock count exceeded");
                    // 更改锁的记录状态
                    setState(nextc);
                    return true;
                }
                return false;
            }
    

#### 2.1.3 acquireQueued方法

addWaiter方法已经在上面讲解了。acquireQueued方法作用就是获取锁，如果没有获取到，就让当前线程阻塞等待。

         /**
         * 想要获取锁的 acquire系列方法，都会这个方法来获取锁
         * 循环通过tryAcquire方法不断去获取锁，如果没有获取成功，
         * 就有可能调用parkAndCheckInterrupt方法，让当前线程阻塞
         * @param node 想要获取锁的节点
         * @param arg
         * @return 返回true，表示在线程等待的过程中，线程被中断了
         */
        final boolean acquireQueued(final Node node, int arg) {
            boolean failed = true;
            try {
                // 表示线程在等待过程中，是否被中断了
                boolean interrupted = false;
                // 通过死循环，直到node节点的线程获取到锁，才返回
                for (;;) {
                    // 获取node的前一个节点
                    final Node p = node.predecessor();
                    // 如果前一个节点是队列头head，并且尝试获取锁成功
                    // 那么当前线程就不需要阻塞等待，继续执行
                    if (p == head && tryAcquire(arg)) {
                        // 将节点node设置为新的队列头
                        setHead(node);
                        // help GC
                        p.next = null;
                        // 不需要调用cancelAcquire方法
                        failed = false;
                        return interrupted;
                    }
                    // 当p节点的状态是Node.SIGNAL时，就会调用parkAndCheckInterrupt方法，阻塞node线程
                    // node线程被阻塞，有两种方式唤醒，
                    // 1.是在unparkSuccessor(Node node)方法，会唤醒被阻塞的node线程，返回false
                    // 2.node线程被调用了interrupt方法，线程被唤醒，返回true
                    // 在这里只是简单地将interrupted = true，没有跳出for的死循环，继续尝试获取锁
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {
                // failed为true，表示发生异常，非正常退出
                // 则将node节点的状态设置成CANCELLED，表示node节点所在线程已取消，不需要唤醒了。
                if (failed)
                    cancelAcquire(node);
            }
        }
    

主要流程：

> 1.  通过for (;;)死循环，直到node节点的线程获取到锁，才跳出循环。
> 2.  获取node节点的前一个节点p。
> 3.  当前一个节点p时CLH队列头节点时，调用tryAcquire方法尝试去获取锁，如果获取成功，就将节点node设置成CLH队列头节点(相当于移除节点node和之前的节点)然后return返回。  
>     注意:只有当node节点的前一个节点是队列头节点时，才会尝试获取锁，所以获取锁是有顺序的，按照添加到CLH队列时的顺序。
> 4.  调用shouldParkAfterFailedAcquire方法，来决定是否要阻塞当前线程。
> 5.  调用parkAndCheckInterrupt方法，阻塞当前线程。
> 6.  如果当前线程发生异常，非正常退出，那么会在finally模块中调用cancelAcquire(node)方法，取消当前节点状态。

> 注意：这里当尝试获取锁失败时，并没有立即阻塞当前线程，但是因为在for (;;)死循环里，会继续循环，方法不会返回。

#### 2.1.4 shouldParkAfterFailedAcquire方法

这个方法的返回值决定是否要阻塞当前线程

        /**
         * 根据前一个节点pred的状态，来判断当前线程是否应该被阻塞
         * @param pred : node节点的前一个节点
         * @param node
         * @return 返回true 表示当前线程应该被阻塞，之后应该会调用parkAndCheckInterrupt方法来阻塞当前线程
         */
        private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            int ws = pred.waitStatus;
            if (ws == Node.SIGNAL)
                // 如果前一个pred的状态是Node.SIGNAL，那么直接返回true，当前线程应该被阻塞
                return true;
            if (ws > 0) {
                // 如果前一个节点状态是Node.CANCELLED(大于0就是CANCELLED)，
                // 表示前一个节点所在线程已经被唤醒了，要从CLH队列中移除CANCELLED的节点。
                // 所以从pred节点一直向前查找直到找到不是CANCELLED状态的节点，并把它赋值给node.prev，
                // 表示node节点的前一个节点已经改变。
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                pred.next = node;
            } else {
                // 此时前一个节点pred的状态只能是0或者PROPAGATE，不可能是CONDITION状态
                // CONDITION(这个是特殊状态，只在condition列表中节点中存在，CLH队列中不存在这个状态的节点)
                // 将前一个节点pred的状态设置成Node.SIGNAL，这样在下一次循环时，就是直接阻塞当前线程
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }
    

我们发现是根据前一个节点的状态，来决定是否阻塞当前线程。而前一个节点状态是在哪里改变的呢？惊奇地发现也是在这个方法中改变的。

> 1.  如果前一个节点状态是Node.SIGNAL，那么直接返回true，阻塞当前线程
> 2.  如果前一个节点状态是Node.CANCELLED(大于0就是CANCELLED)，表示前一个节点所在线程已经被唤醒了，要从CLH队列中移除CANCELLED的节点。所以从pred节点一直向前查找直到找到不是CANCELLED状态的节点。  
>     并把它赋值给node.prev，表示node节点的前一个节点已经改变。在acquireQueued方法中进行下一次循环。
> 3.  不是前面两种状态，那么就将前一个节点状态设置成Node.SIGNAL，表示需要阻塞当前线程，这样再下一次循环时，就会直接阻塞当前线程。

#### 2.1.5 parkAndCheckInterrupt 方法

阻塞当前线程，线程被唤醒后返回当前线程中断状态

        /**
         * 阻塞当前线程，线程被唤醒后返回当前线程中断状态
         */
        private final boolean parkAndCheckInterrupt() {
            // 通过LockSupport.park方法，阻塞当前线程
            LockSupport.park(this);
            // 当前线程被唤醒后，返回当前线程中断状态
            return Thread.interrupted();
        }
    

通过LockSupport.park(this)阻塞当前线程。

#### 2.1.6 cancelAcquire方法

将node节点的状态设置成CANCELLED，表示node节点所在线程已取消，不需要唤醒了。

        // 将node节点的状态设置成CANCELLED，表示node节点所在线程已取消，不需要唤醒了。
        private void cancelAcquire(Node node) {
            // 如果node为null，就直接返回
            if (node == null)
                return;
    
            //
            node.thread = null;
    
            // 跳过那些已取消的节点，在队列中找到在node节点前面的第一次状态不是已取消的节点
            Node pred = node.prev;
            while (pred.waitStatus > 0)
                node.prev = pred = pred.prev;
    
            // 记录pred原来的下一个节点，用于CAS函数更新时使用
            Node predNext = pred.next;
    
            // Can use unconditional write instead of CAS here.
            // After this atomic step, other Nodes can skip past us.
            // Before, we are free of interference from other threads.
            // 将node节点状态设置为已取消Node.CANCELLED;
            node.waitStatus = Node.CANCELLED;
    
            // 如果node节点是队列尾节点，那么就将pred节点设置为新的队列尾节点
            if (node == tail && compareAndSetTail(node, pred)) {
                // 并且设置pred节点的下一个节点next为null
                compareAndSetNext(pred, predNext, null);
            } else {
                // If successor needs signal, try to set pred's next-link
                // so it will get one. Otherwise wake it up to propagate.
                int ws;
                if (pred != head &&
                    ((ws = pred.waitStatus) == Node.SIGNAL ||
                     (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                    pred.thread != null) {
                    Node next = node.next;
                    if (next != null && next.waitStatus <= 0)
                        compareAndSetNext(pred, predNext, next);
                } else {
                    unparkSuccessor(node);
                }
    
                node.next = node; // help GC
            }
        }
    

#### 2.1.7 小结

> 1.  tryAcquire方法尝试获取独占锁，由子类实现
> 2.  acquireQueued方法，方法内采用for死循环，先调用tryAcquire方法，尝试获取锁，如果成功，则跳出循环方法返回。 如果失败，就可能阻塞当前线程。当别的线程锁释放的时候，可能会唤醒这个线程，然后再次进行循环判断，调用tryAcquire方法，尝试获取锁。
> 3.  如果发生异常，该线程被唤醒，所以要取消节点node的状态，因为节点node所在线程不是在阻塞状态了。

> 注：还有其他获取独占锁的方法，例如doAcquireInterruptibly、doAcquireNanos都在文章最后的源码解析中，这里就不做解析了，具体原理都差不多。

### 2.2 释放独占锁的方法

#### 2.2.1 release方法

        // 在独占锁模式下，释放锁的操作
        public final boolean release(int arg) {
            // 调用tryRelease方法，尝试去释放锁，由子类具体实现
            if (tryRelease(arg)) {
                Node h = head;
                // 如果队列头节点的状态不是0，那么队列中就可能存在需要唤醒的等待节点。
                // 还记得我们在acquireQueued(final Node node, int arg)获取锁的方法中，如果节点node没有获取到锁，
                // 那么我们会将节点node的前一个节点状态设置为Node.SIGNAL，然后调用parkAndCheckInterrupt方法
                // 将节点node所在线程阻塞。
                // 在这里就是通过unparkSuccessor方法，进而调用LockSupport.unpark(s.thread)方法，唤醒被阻塞的线程
                if (h != null && h.waitStatus != 0)
                    unparkSuccessor(h);
                return true;
            }
            return false;
        }
    

> 1.  调用tryRelease方法释放锁资源，返回true表示锁资源完全释放了，返回false表示还持有锁资源。
> 2.  如果锁资源完全被释放了，就要唤醒等待锁资源的线程。调用unparkSuccessor方法唤醒一个等待线程  
>     注：CLH队列头节点h为null，表示队列为空，没有节点。节点h的状态是0，表示没有CLH队列中没有被阻塞的线程。

#### 2.2.2 tryRelease方法

         // 尝试去释放当前线程持有的独占锁，立即返回。如果返回true表示释放锁成功
        protected boolean tryRelease(int arg) {
            throw new UnsupportedOperationException();
        }
    

如果子类想实现独占锁，则必须重写这个方法，否则抛出异常。作用是释放当前线程持有的锁，返回true表示已经完全释放锁资源，返回false，表示还持有锁资源。

> 注：对于独占锁来说，同一时间只能有一个线程持有这个锁，但是这个线程可以重复地获取锁，因为被锁住的模块，再次进入另一个被这个锁锁住的模块，是允许的。这个就做可重入性，所以对于可重入的锁释放操作，也需要多次。

下面是ReentrantLock中Sync的tryRelease方法实现

       protected final boolean tryRelease(int releases) {
                // c表示新的锁的记录状态
                int c = getState() - releases;
                // 如果当前线程不是独占锁的线程，就抛出IllegalMonitorStateException异常
                if (Thread.currentThread() != getExclusiveOwnerThread())
                    throw new IllegalMonitorStateException();
                // 标志是否可以释放锁
                boolean free = false;
                // 当新的锁的记录状态为0时，表示可以释放锁
                if (c == 0) {
                    free = true;
                    // 设置独占锁的线程为null
                    setExclusiveOwnerThread(null);
                }
                setState(c);
                return free;
            }
    

#### 2.2.3 unparkSuccessor方法

       // 唤醒node节点的下一个非取消状态的节点所在线程(即waitStatus<=0)
        private void unparkSuccessor(Node node) {
            // 获取node节点的状态
            int ws = node.waitStatus;
            // 如果小于0，就将状态重新设置为0，表示这个node节点已经完成了
            if (ws < 0)
                compareAndSetWaitStatus(node, ws, 0);
    
            // 下一个节点
            Node s = node.next;
            // 如果下一个节点为null，或者状态是已取消，那么就要寻找下一个非取消状态的节点
            if (s == null || s.waitStatus > 0) {
                // 先将s设置为null，s不是非取消状态的节点
                s = null;
                // 从队列尾向前遍历，直到遍历到node节点
                for (Node t = tail; t != null && t != node; t = t.prev)
                    // 因为是从后向前遍历，所以不断覆盖找到的值，这样才能得到node节点后下一个非取消状态的节点
                    if (t.waitStatus <= 0)
                        s = t;
            }
            // 如果s不为null，表示存在非取消状态的节点。那么调用LockSupport.unpark方法，唤醒这个节点的线程
            if (s != null)
                LockSupport.unpark(s.thread);
        }
    

这个方法的作用是唤醒node节点的下一个非取消状态的节点所在线程。

> 1.  将node节点的状态设置为0
> 2.  寻找到下一个非取消状态的节点s
> 3.  如果节点s不为null，则调用LockSupport.unpark(s.thread)方法唤醒s所在线程。  
>     注：唤醒线程也是有顺序的，就是添加到CLH队列线程的顺序。

#### 2.2.4 小结

> 1.  调用tryRelease方法去释放当前持有的锁资源。
> 2.  如果完全释放了锁资源，那么就调用unparkSuccessor方法，去唤醒一个等待锁的线程。

三. 共享锁
------

共享锁与独占锁相比，共享锁可能被多个线程共同持有

### 3.1 获取共享锁的方法

#### 3.1.1 acquireShared方法

        // 获取共享锁
        public final void acquireShared(int arg) {
            // 尝试去获取共享锁，如果返回值小于0表示获取共享锁失败
            if (tryAcquireShared(arg) < 0)
                // 调用doAcquireShared方法去获取共享锁
                doAcquireShared(arg);
        }
    

调用tryAcquireShared方法尝试获取共享锁，如果返回值小于0表示获取共享锁失败.则继续调用doAcquireShared方法获取共享锁。

#### 3.1.2 tryAcquireShared方法

         // 尝试去获取共享锁，立即返回。返回值大于等于0，表示获取共享锁成功
        protected int tryAcquireShared(int arg) {
            throw new UnsupportedOperationException();
        }
    

如果子类想实现共享锁，则必须重写这个方法，否则抛出异常。作用是尝试获取共享锁，返回值大于等于0，表示获取共享锁成功。

下面是ReentrantReadWriteLock中Sync的tryReleaseShared方法实现,这个我们会在ReentrantReadWriteLock章节中重点介绍的。

          protected final boolean tryReleaseShared(int unused) {
                Thread current = Thread.currentThread();
                // 当前线程是第一个获取读锁(共享锁)的线程
                if (firstReader == current) {
                    // 将firstReaderHoldCount减一，如果就是1，那么表示该线程需要释放读锁(共享锁)，
                    // 将firstReader设置为null
                    if (firstReaderHoldCount == 1)
                        firstReader = null;
                    else
                        firstReaderHoldCount--;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    // 获取当前线程的HoldCounter变量
                    if (rh == null || rh.tid != getThreadId(current))
                        rh = readHolds.get();
                    // 将rh变量的count减一，
                    int count = rh.count;
                    if (count <= 1) {
                        readHolds.remove();
                        // count <= 0表示当前线程就没有获取到读锁(共享锁)，这里释放就抛出异常。
                        if (count <= 0)
                            throw unmatchedUnlockException();
                    }
                    --rh.count;
                }
                for (;;) {
                    int c = getState();
                    // 因为读锁是利用高16位储存的，低16位的数据是要屏蔽的，
                    // 所以这里减去SHARED_UNIT(65536)，相当于减一
                    // 表示一个读锁已经释放
                    int nextc = c - SHARED_UNIT;
                    // 利用CAS函数重新设置state值
                    if (compareAndSetState(c, nextc))
                        return nextc == 0;
                }
            }
    

#### 3.1.3 doAcquireShared方法

        /**
         * 获取共享锁，获取失败，则会阻塞当前线程，直到获取共享锁返回
         * @param arg the acquire argument
         */
        private void doAcquireShared(int arg) {
            // 为当前线程创建共享锁节点node
            final Node node = addWaiter(Node.SHARED);
            boolean failed = true;
            try {
                boolean interrupted = false;
                for (;;) {
                    final Node p = node.predecessor();
                    // 如果节点node前一个节点是同步队列头节点。就会调用tryAcquireShared方法尝试获取共享锁
                    if (p == head) {
                        int r = tryAcquireShared(arg);
                        // 如果返回值大于0，表示获取共享锁成功
                        if (r >= 0) {
                            setHeadAndPropagate(node, r);
                            p.next = null; // help GC
                            if (interrupted)
                                selfInterrupt();
                            failed = false;
                            return;
                        }
                    }
                    // 如果节点p的状态是Node.SIGNAL，就是调用parkAndCheckInterrupt方法阻塞当前线程
                    if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                        interrupted = true;
                }
            } finally {
                 // failed为true，表示发生异常，
                // 则将node节点的状态设置成CANCELLED，表示node节点所在线程已取消，不需要唤醒了
                if (failed)
                    cancelAcquire(node);
            }
        }
    

这个方法与独占锁的acquireQueued方法相比较，不同的有三点：

> 1.  doAcquireShared方法，调用addWaiter(Node.SHARED)方法，为当前线程创建一个共享模式的节点node。而acquireQueued方法是由外部传递来的。
> 2.  doAcquireShared方法没有返回值，acquireQueued方法会返回布尔类型的值，是当前线程中断标志位值
> 3.  最大的区别是重新设置CLH队列头的方法不一样。doAcquireShared方法调用setHeadAndPropagate方法，而acquireQueued方法调用setHead方法。

#### 3.1.4 setHeadAndPropagate方法

         // 重新设置CLH队列头，如果CLH队列头的下一个节点为null或者共享模式，
        // 那么就要唤醒共享锁上等待的线程
        private void setHeadAndPropagate(Node node, int propagate) {
            Node h = head;
            // 设置新的同步队列头head
            setHead(node);
            // 如果propagate大于0，
            if (propagate > 0 || h == null || h.waitStatus < 0 ||
                (h = head) == null || h.waitStatus < 0) {
                // 获取新的CLH队列头的下一个节点s
                Node s = node.next;
                // 如果节点s是空或者共享模式节点，那么就要唤醒共享锁上等待的线程
                if (s == null || s.isShared())
                    doReleaseShared();
            }
        }
    

### 3.2 释放共享锁的方法

#### 3.2.1 releaseShared方法

         // 释放共享锁
        public final boolean releaseShared(int arg) {
            // 尝试释放共享锁
            if (tryReleaseShared(arg)) {
                // 唤醒等待共享锁的线程
                doReleaseShared();
                return true;
            }
            return false;
        }
    

#### 3.2.2 tryReleaseShared方法

         // 尝试去释放共享锁
        protected boolean tryReleaseShared(int arg) {
            throw new UnsupportedOperationException();
        }
    

如果子类想实现共享锁，则必须重写这个方法，否则抛出异常。作用是释放当前线程持有的锁，返回true表示已经完全释放锁资源，返回false，表示还持有锁资源。

#### 3.2.3 doReleaseShared方法

         // 会唤醒等待共享锁的线程
        private void doReleaseShared() {
            for (;;) {
                // 将同步队列头赋值给节点h
                Node h = head;
                // 如果节点h不为null，且不等于同步队列尾
                if (h != null && h != tail) {
                    // 得到节点h的状态
                    int ws = h.waitStatus;
                    // 如果状态是Node.SIGNAL，就要唤醒节点h后继节点的线程
                    if (ws == Node.SIGNAL) {
                        // 将节点h的状态设置成0，如果设置失败，就继续循环，再试一次。
                        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                            continue;            // loop to recheck cases
                        // 唤醒节点h后继节点的线程
                        unparkSuccessor(h);
                    }
                    // 如果节点h的状态是0，就设置ws的状态是PROPAGATE。
                    else if (ws == 0 &&
                             !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                        continue;                // loop on failed CAS
                }
                // 如果同步队列头head节点发生改变，继续循环，
                // 如果没有改变，就跳出循环
                if (h == head)
                    break;
            }
        }
    

四. Condition条件
--------------

Condition是为了实现线程之间相互等待的问题。注意Condition对象只能在独占锁中才能使用。

> 考虑一下情况，有两个线程，生产者线程，消费者线程。当消费者线程消费东西时，发现没有东西，这时它就要等待，让生产者线程生产东西后，在通知它消费。  
> 因为操作的是同一个资源，所以要加锁，防止多线程冲突。而锁在同一时间只能有一个线程持有，所以消费者在让线程等待前，必须释放锁，且唤醒另一个等待锁的线程。  
> 那么在AQS中Condition条件又是如何实现的呢？
> 
> 1.  首先内部存在一个Condition队列，存储着所有在此Condition条件等待的线程。
> 2.  await系列方法：让当前持有锁的线程释放锁，并唤醒一个在CLH队列上等待锁的线程，再为当前线程创建一个node节点，插入到Condition队列(注意不是插入到CLH队列中)
> 3.  signal系列方法：其实这里没有唤醒任何线程，而是将Condition队列上的等待节点插入到CLH队列中，所以当持有锁的线程执行完毕释放锁时，就会唤醒CLH队列中的一个线程，这个时候才会唤醒线程。

### 4.1 await系列方法

#### 4.1.1 await方法

         /**
             * 让当前持有锁的线程阻塞等待，并释放锁。如果有中断请求，则抛出InterruptedException异常
             * @throws InterruptedException
             */
            public final void await() throws InterruptedException {
                // 如果当前线程中断标志位是true，就抛出InterruptedException异常
                if (Thread.interrupted())
                    throw new InterruptedException();
                // 为当前线程创建新的Node节点，并且将这个节点插入到Condition队列中了
                Node node = addConditionWaiter();
                // 释放当前线程占有的锁，并唤醒CLH队列一个等待线程
                int savedState = fullyRelease(node);
                int interruptMode = 0;
                // 如果节点node不在同步队列中(注意不是Condition队列)
                while (!isOnSyncQueue(node)) {
                    // 阻塞当前线程,那么怎么唤醒这个线程呢？
                    // 首先我们必须调用signal或者signalAll将这个节点node加入到同步队列。
                    // 只有这样unparkSuccessor(Node node)方法，才有可能唤醒被阻塞的线程
                    LockSupport.park(this);
                    // 如果当前线程产生中断请求，就跳出循环
                    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                        break;
                }
                // 如果节点node已经在同步队列中了，获取同步锁，只有得到锁才能继续执行，否则线程继续阻塞等待
                if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                    interruptMode = REINTERRUPT;
                // 清除Condition队列中状态不是Node.CONDITION的节点
                if (node.nextWaiter != null)
                    unlinkCancelledWaiters();
                // 是否要抛出异常，或者发出中断请求
                if (interruptMode != 0)
                    reportInterruptAfterWait(interruptMode);
            }
    

方法流程：

> 1.  addConditionWaiter方法：为当前线程创建新的Node节点，并且将这个节点插入到Condition队列中了
> 2.  fullyRelease方法：释放当前线程占有的锁，并唤醒CLH队列一个等待线程
> 3.  isOnSyncQueue 方法：如果返回false，表示节点node不在CLH队列中，即没有调用过 signal系列方法，所以调用LockSupport.park(this)方法阻塞当前线程。
> 4.  如果跳出while循环，表示节点node已经在CLH队列中，那么调用acquireQueued方法去获取锁。
> 5.  清除Condition队列中状态不是Node.CONDITION的节点

#### 4.1.2 addConditionWaiter方法

为当前线程创建新的Node节点，并且将这个节点插入到Condition队列中了

        private Node addConditionWaiter() {
                Node t = lastWaiter;
                // 如果Condition队列尾节点的状态不是Node.CONDITION
                if (t != null && t.waitStatus != Node.CONDITION) {
                    // 清除Condition队列中，状态不是Node.CONDITION的节点，
                    // 并且可能会重新设置firstWaiter和lastWaiter
                    unlinkCancelledWaiters();
                    // 重新将Condition队列尾赋值给t
                    t = lastWaiter;
                }
                // 为当前线程创建一个状态为Node.CONDITION的节点
                Node node = new Node(Thread.currentThread(), Node.CONDITION);
                // 如果t为null，表示Condition队列为空，将node节点赋值给链表头
                if (t == null)
                    firstWaiter = node;
                else
                    // 将新节点node插入到Condition队列尾
                    t.nextWaiter = node;
                // 将新节点node设置为新的Condition队列尾
                lastWaiter = node;
                return node;
            }
    

#### 4.1.3 fullyRelease方法

释放当前线程占有的锁，并唤醒CLH队列一个等待线程

        /**
         * 释放当前线程占有的锁，并唤醒CLH队列一个等待线程
         * 如果失败就抛出异常，设置node节点的状态是Node.CANCELLED
         * @return
         */
        final int fullyRelease(Node node) {
            boolean failed = true;
            try {
                int savedState = getState();
                // 释放当前线程占有的锁
                if (release(savedState)) {
                    failed = false;
                    return savedState;
                } else {
                    throw new IllegalMonitorStateException();
                }
            } finally {
                if (failed)
                    node.waitStatus = Node.CANCELLED;
            }
        }
    

#### 4.1.4 isOnSyncQueue

节点node是不是在CLH队列中

        // 节点node是不是在CLH队列中
        final boolean isOnSyncQueue(Node node) {
            // 如果node的状态是Node.CONDITION，或者node没有前一个节点prev，
            // 那么返回false，节点node不在同步队列中
            if (node.waitStatus == Node.CONDITION || node.prev == null)
                return false;
            // 如果node有下一个节点next，那么它一定在同步队列中
            if (node.next != null) // If has successor, it must be on queue
                return true;
            // 从同步队列中查找节点node
            return findNodeFromTail(node);
        }
    
        // 在同步队列中从后向前查找节点node，如果找到返回true，否则返回false
        private boolean findNodeFromTail(Node node) {
            Node t = tail;
            for (;;) {
                if (t == node)
                    return true;
                if (t == null)
                    return false;
                t = t.prev;
            }
        }
    

#### 4.1.5 acquireQueued方法

获取独占锁，这个在独占锁章节已经说过

#### 4.1.6 unlinkCancelledWaiters 方法

清除Condition队列中状态不是Node.CONDITION的节点

        private void unlinkCancelledWaiters() {
                // condition队列头赋值给t
                Node t = firstWaiter;
                // 这个trail节点，只是起辅助作用
                Node trail = null;
                while (t != null) {
                    //得到下一个节点next。当节点是condition时候，nextWaiter表示condition队列的下一个节点
                    Node next = t.nextWaiter;
                    // 如果节点t的状态不是CONDITION，那么该节点就要从condition队列中移除
                    if (t.waitStatus != Node.CONDITION) {
                        // 将节点t的nextWaiter设置为null
                        t.nextWaiter = null;
                        // 如果trail为null，表示原先的condition队列头节点实效，需要设置新的condition队列头
                        if (trail == null)
                            firstWaiter = next;
                        else
                            // 将节点t从condition队列中移除，因为改变了引用的指向，从condition队列中已经找不到节点t了
                            trail.nextWaiter = next;
                        // 如果next为null，表示原先的condition队列尾节点也实效，重新设置队列尾节点
                        if (next == null)
                            lastWaiter = trail;
                    }
                    else
                        // 遍历到的有效节点
                        trail = t;
                    // 将next赋值给t，遍历完整个condition队列
                    t = next;
                }
            }
    

#### 4.1.7 reportInterruptAfterWait方法

        /**
             * 如果interruptMode是THROW_IE，就抛出InterruptedException异常
             * 如果interruptMode是REINTERRUPT，则当前线程再发出中断请求
             * 否则就什么都不做
             */
            private void reportInterruptAfterWait(int interruptMode)
                throws InterruptedException {
                if (interruptMode == THROW_IE)
                    throw new InterruptedException();
                else if (interruptMode == REINTERRUPT)
                    selfInterrupt();
            }
    

### 4.2 signal系列方法

#### 4.2.1 signal方法

           // 如果condition队列不为空，将condition队列头节点插入到同步队列中
            public final void signal() {
                // 如果当前线程不是独占锁线程，就抛出IllegalMonitorStateException异常
                if (!isHeldExclusively())
                    throw new IllegalMonitorStateException();
    
                // 将Condition队列头赋值给节点first
                Node first = firstWaiter;
                if (first != null)
                    //  将Condition队列中的first节点插入到CLH队列中
                    doSignal(first);
            }
    

如果condition队列不为空，就调用doSignal方法将condition队列头节点插入到CLH队列中。

#### 4.2.2 doSignal方法

            // 将Condition队列中的first节点插入到CLH队列中
            private void doSignal(Node first) {
                do {
                    // 原先的Condition队列头节点取消，所以重新赋值Condition队列头节点
                    // 如果新的Condition队列头节点为null，表示Condition队列为空了
                    // ，所以也要设置Condition队列尾lastWaiter为null
                    if ( (firstWaiter = first.nextWaiter) == null)
                        lastWaiter = null;
                    // 取消first节点nextWaiter引用
                    first.nextWaiter = null;
                } while (!transferForSignal(first) &&
                         (first = firstWaiter) != null);
            }
    

为什么使用while循环，因为只有是Node.CONDITION状态的节点才能插入CLH队列，如果不是这个状态，那么循环Condition队列下一个节点。

#### 4.2.3 transferForSignal方法

          // 返回true表示节点node插入到同步队列中，返回false表示节点node没有插入到同步队列中
        final boolean transferForSignal(Node node) {
            // 如果节点node的状态不是Node.CONDITION，或者更新状态失败，
            // 说明该node节点已经插入到同步队列中，所以直接返回false
            if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
                return false;
    
            // 将节点node插入到同步队列中，p是原先同步队列尾节点，也是node节点的前一个节点
            Node p = enq(node);
            int ws = p.waitStatus;
            // 如果前一个节点是已取消状态，或者不能将它设置成Node.SIGNAL状态。
            // 就说明节点p之后也不会发起唤醒下一个node节点线程的操作，
            // 所以这里直接调用 LockSupport.unpark(node.thread)方法，唤醒节点node所在线程
            if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
                LockSupport.unpark(node.thread);
            return true;
        }
    

> 1.  状态不是 Node.CONDITION的节点，是不能从Condition队列中插入到CLH队列中。直接返回false
> 2.  调用enq方法，将节点node插入到同步队列中，p是原先同步队列尾节点，也是node节点的前一个节点
> 3.  如果前一个节点是已取消状态，或者不能将它设置成Node.SIGNAL状态。那么就要LockSupport.unpark(node.thread)方法唤醒node节点所在线程。

#### 4.2.4 signalAll 方法

         // 将condition队列中所有的节点都插入到同步队列中
            public final void signalAll() {
                if (!isHeldExclusively())
                    throw new IllegalMonitorStateException();
                Node first = firstWaiter;
                if (first != null)
                    doSignalAll(first);
            }
    

#### 4.2.5 doSignalAll方法

        /**
             * 将condition队列中所有的节点都插入到同步队列中
             * @param first condition队列头节点
             */
            private void doSignalAll(Node first) {
                // 表示将condition队列设置为空
                lastWaiter = firstWaiter = null;
                do {
                    // 得到condition队列的下一个节点
                    Node next = first.nextWaiter;
                    first.nextWaiter = null;
                    // 将节点first插入到同步队列中
                    transferForSignal(first);
                    first = next;
                    // 循环遍历condition队列中所有的节点
                } while (first != null);
            }
    

循环遍历整个condition队列，调用transferForSignal方法，将节点插入到CLH队列中。

### 4.3 小结

Condition只能使用在独占锁中。它内部有一个Condition队列记录所有在Condition条件等待的线程(即就是调用await系列方法后等待的线程).  
await系列方法：会让当前线程释放持有的锁，并唤醒在CLH队列上的一个等待锁的线程，再将当前线程插入到Condition队列中(注意不是CLH队列)  
signal系列方法：并不是唤醒线程，而是将Condition队列中的节点插入到CLH队列中。

总结
--

使用AQS类来实现独占锁和共享锁：

> 1.  内部有一个CLH队列，用来记录所有等待锁的线程
> 2.  通过 acquire系列方法用来获取独占锁，获取失败，则阻塞当前线程
> 3.  通过release方法用来释放独占锁，释放成功，则会唤醒一个等待独占锁的线程。
> 4.  通过acquireShared系列方法用来获取共享锁。
> 5.  通过releaseShared方法用来释放共享锁。
> 6.  通过Condition来实现线程之间相互等待的。

示例
--

    import java.util.concurrent.locks.AbstractQueuedSynchronizer;
    import java.util.concurrent.locks.Lock;
    import java.util.concurrent.locks.ReentrantLock;
    
    // 实现一个可重入的独占锁， 必须复写tryAcquire和tryRelease方法
    class Sync extends AbstractQueuedSynchronizer {
        // 尝试获取独占锁， 利用锁的获取次数state属性
        @Override
        protected boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            // 获取锁的记录状态state
            int c = getState();
            // 如果c==0表示当前锁是空闲的
            if (c == 0) {
                // 通过CAS原子操作方式设置锁的状态，如果为true，表示当前线程获取的锁，
                // 为false，锁的状态被其他线程更改，当前线程获取的锁失败
                if (compareAndSetState(0, acquires)) {
                    // 设置当前线程为独占锁的线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 判断当前线程是不是独占锁的线程，因为是可重入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                // 更改锁的记录状态
                setState(nextc);
                return true;
            }
            return false;
        }
    
        // 释放持有的独占锁， 因为是可重入锁，所以只有当c等于0的时候，表示当前持有锁的完全释放了锁。
        @Override
        protected boolean tryRelease(int releases) {
            // c表示新的锁的记录状态
            int c = getState() - releases;
            // 如果当前线程不是独占锁的线程，就抛出IllegalMonitorStateException异常
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            // 标志是否可以释放锁
            boolean free = false;
            // 当新的锁的记录状态为0时，表示可以释放锁
            if (c == 0) {
                free = true;
                // 设置独占锁的线程为null
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
    }
    
    public class AQSTest {
    
        public static void newThread(Sync sync, String name, int time) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println("线程"+Thread.currentThread().getName()+" 开始运行，准备获取锁");
                    // 通过acquire方法，获取锁，如果没有获取到，就等待
                    sync.acquire(1);
                    try {
                        System.out.println("====线程"+Thread.currentThread().getName()+" 在run方法中获取了锁");
                        lockAgain();
                        try {
                            Thread.sleep(time);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    } finally {
                        System.out.println("----线程"+Thread.currentThread().getName()+" 在run方法中释放了锁");
                        sync.release(1);
                    }
                }
    
                private void lockAgain() {
                    sync.acquire(1);
                    try {
                        System.out.println("====线程"+Thread.currentThread().getName()+"  在lockAgain方法中再次获取了锁");
                        try {
                            Thread.sleep(10);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    } finally {
                        System.out.println("----线程"+Thread.currentThread().getName()+" 在lockAgain方法中释放了锁");
                        sync.release(1);
                    }
                }
            },name).start();
        }
    
        public static void main(String[] args) {
            Sync sync = new Sync();
            newThread(sync, "t1111", 1000);
            newThread(sync, "t2222", 1000);
            newThread(sync, "t3333", 1000);
        }
    }
    

利用AbstractQueuedSynchronizer简单实现一个可重入的独占锁。

> 1.  要实现独占锁，必须重写tryAcquire和tryRelease方法。否则在获取锁和释放锁的时候，会抛出异常。
> 2.  直接的调用AQS类的acquire(1)和release(1)方法获取锁和释放锁。

输出结果是

    线程t1111 开始运行，准备获取锁
    ====线程t1111 在run方法中获取了锁
    ====线程t1111  在lockAgain方法中再次获取了锁
    线程t2222 开始运行，准备获取锁
    线程t3333 开始运行，准备获取锁
    ----线程t1111 在lockAgain方法中释放了锁
    ----线程t1111 在run方法中释放了锁
    ====线程t2222 在run方法中获取了锁
    ====线程t2222  在lockAgain方法中再次获取了锁
    ----线程t2222 在lockAgain方法中释放了锁
    ----线程t2222 在run方法中释放了锁
    ====线程t3333 在run方法中获取了锁
    ====线程t3333  在lockAgain方法中再次获取了锁
    ----线程t3333 在lockAgain方法中释放了锁
    ----线程t3333 在run方法中释放了锁
    

