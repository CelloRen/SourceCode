# AQS源码分析

## 底层的数据结构：

一个CLH链表

```java
static final class Node {
        // 共享模式
        static final Node SHARED = new Node();
        // 独占模式
        static final Node EXCLUSIVE = null;

        // 线程状态为：被取消，timeout or 中断
        static final int CANCELLED =  1;
        // 线程状态为：待唤醒，head唤醒后续节点
        static final int SIGNAL    = -1;
        // 线程状态为：待condition.signal唤醒
        static final int CONDITION = -2;
        // 共享模式中的线程状态：唤醒会传播
        static final int PROPAGATE = -3;

        // 上述的线程状态，初始值为0
        // waitStatus > 0表示node被取消，waitStatus < 0表示有序等待
        volatile int waitStatus;

        volatile Node prev;
  
        volatile Node next;

        volatile Thread thread;

        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        // 返回当前node的prev node
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



## 涉及到的CAS操作：

```java
unsafe.compareAndSwapObject(object, offset, expect, update)
```

这个操作是CAS，检查object对象的offset偏移量的内存地址是否为expect，是就将其更新为update（获取offset利用了反射）

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    // CAS head field. Used only by enq.
    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    // CAS tail field. Used only by enq.
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    // CAS waitStatus field of a node.
    private static final boolean compareAndSetWaitStatus(Node node,int expect,int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset, expect, update);
    }

    // CAS next field of a node.
    private static final boolean compareAndSetNext(Node node,Node expect,Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }
```



## 主要的入口方法：

独占式获取：

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

tryAcquire是具体的子类实现的（尝试获取），获取成功则返回true，方法结束（false && x时，会直接短路）

失败则：addWaiter将其放在链表的尾部

```java
    private Node addWaiter(Node mode) {
        // 将线程封装为Node，mode为独占式
        Node node = new Node(Thread.currentThread(), mode);
        // 尝试快速放在tail处
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // 这里的CAS是将tail内存处替换为node
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        // 自旋，直到CAS成功
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

然后acquireQueued

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

