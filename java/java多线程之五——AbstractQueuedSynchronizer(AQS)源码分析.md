>本文基于java version "1.8.0_77"

阅读本文章之前，你需要了解[LockSupport](http://www.jianshu.com/p/8bdf56ee9e32)中相关方法的介绍。
阅读本篇文章，请对照源码阅读，否则可能云里雾里不知所云。
# 简介
AbstractQueuedSynchronizer：译为：队列同步器（以下简称AQS），可以看到这是一个抽象类。有大名鼎鼎的并发大师Doug Lea设计：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-1f1beac67d19630e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

并发包中很多Lock都是通过继承AQS实现的（ReentrantLock、
ReentrantReadWriteLock和CountDownLatch等），AQS中封装了实现锁的具体操作，其子类继承AQS后，可以轻松的调用AQS的相应方法来实现同步状态的管理同步状态，线程的排队，等待以及唤醒等操作。
子类可以重写的方法如下：
* ```protected boolean tryAcquire(int arg)```独占式的获取同步状态，使用CAS设置同步状态
* ```protected boolean tryRelease(int arg)```独占式的释放同步状态
* ``` protected int tryAcquireShared(int arg)```共享式的获取同步状态，返回大于等于0的值，表示获取成功，否则失败
* ``` protected boolean tryReleaseShared(int arg)```
共享式的释放同步状态
* ``` protected boolean isHeldExclusively()```判断当前是否被当前线程锁独占

# 构成
![image.png](http://upload-images.jianshu.io/upload_images/1583231-28c03151254aef8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

如上图，AQS中定义了一个volatile整数状态信息，我们可以通过``` getState()```,```setState(int newState)```,```compareAndSetState(int expect,int update)）```等protected方法进行操作这一状态信息。例如：ReentrantLock中用它来表示所有线程呢个已经重复获取该锁的次数，Semaphore用它来表示剩余的许可数量，FutureTask用它来表示任务的状态（未开始，正在运行，已结束，已取消等）。

AQS是由一个同步队列（FIFO双向队列）来管理同步状态的，如果线程获取同步状态失败，AQS会将当前线程以及等待状态信息构造成一个节点（Node）加入到同步队列中，同时阻塞当前线程；当同步状态状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

![image.png](http://upload-images.jianshu.io/upload_images/1583231-5a34f82bab7a4a94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 准备工作
在跟着源码走流程之前，我们先了一下以下几个需要用到的概念：
## AQS.Node

队列示意图如下:
![image.png](http://upload-images.jianshu.io/upload_images/1583231-376e68f0f37ee9f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)



每个Node节点都是一个```自旋锁```：在阻塞时不断循环读取状态变量，当前驱节点释放同步对象使用权后，跳出循环，执行同步代码。我们在接下来的代码分析中，也能够看到通过死循环来达到自旋这一目的。

我们看一下Node节点类的几个关键属性（不必记住，下面用到的时候，再回来看即可）：

#### MODE（两个）
![](http://upload-images.jianshu.io/upload_images/1583231-ddc292eb9cf9dd9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
两种Mode，用于创建Node时的构造函数使用。在```private Node addWaiter(Node mode)```这一方法调用的时候传入，用于想等待队列中添加节点。

####  volatile int waitStatus
手机是```waitStatus```,用来表示当前节点的状态。其取值范围如下：
* ```static final int CANCELLED = 1```;表示节点的线程是已被取消的。当前节点由于超时或者被中断而被取消。一旦节点被取消后，那么它的状态值不在会被改变，且当前节点的线程不会再次被阻塞。

* ```static final int SIGNAL= -1```;表示当前节点的后继节点的线程需要被唤醒。```当前节点的后继节点```已经 (或即将)被阻塞（通过LockSupport.park()） , 所以当 当前节点释放或则被取消时候，一定要unpark它的后继节点。为了避免竞争，获取方法一定要首先设置node为signal，然后再次重新调用获取方法，如果失败，则阻塞。

* ```static final int CONDITION = -2;```表示线程正在等待某个条件。表示当前节点正在条件队列（AQS下的ConditionObject里也维护了个队列）中，在从conditionObject队列转移到同步队列前，它不会在同步队列（AQS下的队列）中被使用。当成功转移后，该节点的状态值将由CONDITION设置为0。

* ```static final int PROPAGATE = -3;```表示下一个共享模式的节点应该无条件的传播下去。共享模式下的释放操作应该被传播到其他节点。该状态值在doReleaseShared方法中被设置的。

* ```0``` 以上都不是。

可以看到，非负数值（0和已经取消）意味着该节点不需要被唤醒。所以，大多数代码中不需要检查该状态值的确定值,只需要根据正负值来判断即可对于一个正常的Node，他的waitStatus初始化值时0。对于一个condition队列中的Node，他的初始化值时CONDITION。如果想要修改这个值，可以使用AQS提供CAS进行修改。（方法：``` boolean 
 compareAndSetWaitStatus(Node node, int expect,int update)```）

####   volatile Node prev
用于链接当前节点的前驱节点，当前节点依赖前驱节点来检测waitStatus，前驱节点是在当前节点入队时候被设置的。为了提高GC效率，在当前节点出队时候会把前驱节点设置为null。而且，在取消前驱节点的时候，则会while循环直到找到一个非取消（cancelled）的节点，由于头节点永远不会是取消状态，所以我们一定可以找到非取消状态的前置节点。

####  volatile Node next;
用于链接当前节点的后继节点，在当前节点释放时候会唤醒后继节点。在一个当前节点入队的时候，会先设置当前节点的prev，而不会立即设置前置节点的next。而是用CAS替换了tail之后才设置前置节点的next。（方法Node addWaiter(Node mode)）

#### Node nextWaiter
用来串联条件队列，连接到下一个在条件上等待的结点或是特殊的值SHARED。因为条件队列只在独占模式下持有时访问，我们只需要一个简单的链表队列来持有在条件上等待的结点。再然后它们会被转移到同步队列（AQS队列）再次重新获取。由于条件队列只能在独占模式下使用，所以我们要表示共享模式的节点的话只要使用特殊值SHARED来标明即可。

## 辅助方法分析（供查阅）
####  shouldParkAfterFailedAcquire
这个方法是信号控制（waitStatus）的核心。在获取同步状态失败，生成Node并加入队列中后，用于检查和更新结点的状态。返回true表示当前节点应该被阻塞。
````
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * 前驱节点如果状态如果为SIGNAL。表明当前节点应被阻塞，等待唤醒（参见上文的SIGNAL状态）
             * 则返回true，然后park挂起线程
             */
            return true;
        if (ws > 0) {
            /*
             * 前驱节点状态值大于0（只有一个取值1），表示前驱节点已经取消
             * 此时应该丢弃前驱节点，而继续寻找前驱节点的前驱节点，（见下图）
             * 这里使用while循环查找前驱节点，并将当前节点的prev属性设置为找到的新的节点。（下图步骤1）
             * 并将新的前驱节点的后继节点设置为当前节点（下图步骤2）
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 排除以上SIGNAL（-1）和>0（1）两种情况
             * 现在是前驱节点的waitStatus为0或PROPAGATE（-3）的情况（不考虑CONDITION的情况）
             * 这时候表明前驱节点需要重新设置waitStatus
             * 这样在下一轮循环中，就可以判断前驱节点的SIGNAL而阻塞park当前节点，以便于等待前驱节点的unpark（比如：shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()）
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
````
如图：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-1eda1407f0e6d41f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### parkAndCheckInterrupt

![](http://upload-images.jianshu.io/upload_images/1583231-d424e94f17326da9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

与上面的```shouldParkAfterFailedAcquire```中联合调用
>（shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()）

通过```shouldParkAfterFailedAcquire```方法获取到可用的前驱节点，并设置前驱节点的WaitStatus值为SIGNAL，进而在此方法中将当前线程park（阻塞等待）。线程醒了之后，检查线程是否被重点，并将结果返回。


#### cancelAcquire
上面讲到，每一个NODE节点都是一个自旋锁，都在不断进行死循环自旋，当自旋过程中发生异常而无法获得锁，就需要取消节点。
需要做的是：
* 清空node节点中的引用
* node出队：剔除当前节点，打断next和prev引用。分为三种情况：1. node是tail    2. node既不是tail，也不是head的后继节点  3.  node是head的后继节点
源码分析如下：
````
 private void cancelAcquire(Node node) {
        // 如果node为空，忽略，直接返回
        if (node == null)
            return;
        
        //将thread引用置空
        node.thread = null;

        // 跳过取消的（cancelled）的前置节点，找到一个有效的前驱节点，如上面分析过的shouldParkAfterFailedAcquire
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // 拿到前驱节点的后继节点
        Node predNext = pred.next;

        // 将节点的状态值设为已取消，这样，其他节点就可以跳过本节点，而不受其他线程的干扰
        node.waitStatus = Node.CANCELLED;

        // 情况1：如果当前节点是尾节点，CAS替换tail字段的引用为为前驱节点
        // 成功之后，CAS将前驱节点的后继节点置空
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // 情况2：如果当前节点不是tail，而前驱节点又不是head
            // 则尝试CAS将前驱节点的waitStatus标记为SIGNAL（表示前驱节点的后继节点需要唤醒）
            // 设置成功之后，CAS将前驱节点的后继节点设置为当前节点的后继节点（将当前节点剔除）
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
            // 情况3：如果node是head的后继节点，则直接唤醒node的后继节点
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
````
如上：
情况1：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-aff10a5e985bb573.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)
* 1：compareAndSetTail(node, pred) 替换tail的引用
* 2：compareAndSetNext(pred, predNext, null); 将pred的next置空
--------------------------------
情况2：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-2b6567a8dc61dd40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

* compareAndSetNext(pred, predNext, next); 将前驱节点的next指向后继节点。后继节点的prev将在前面讲过的shouldParkAfterFailedAcquire进行添加。
--------------------------------
情况3
下面将分析unparkSuccessor方法
--------------------------------
#### unparkSuccessor
用于唤醒当前节点的后继节点。
```
private void unparkSuccessor(Node node) {
       // 将当前节点的状态重置
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        // 拿到后继节点 ，如果后继机节点是空或标记为取消（cancelled）
        // 开是循环获取后继的可用节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // LockSupport唤醒下一个节点
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
上文中寻找下一个可用节点的时候，可以看到不是head->tail寻找，而是tail->head倒序寻找，这是因为：通过上面代码可以看到，只有在当前节点node的后继节点为nul的时候，才会执行循环寻找后面的可用后继节点。注意此处：```后继节点已经为null了```，故只能从尾部向前遍历，找到第一个可用节点。

差不多就这些了，下面我们进入正题，探讨一下获取同步化状态的流程。
# -----------------------------------------------------
# 源码分析
### 独占式获取同步状态
上源码：
![](http://upload-images.jianshu.io/upload_images/1583231-48c676e2cb52cc0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

首先tryAcquire(arg)，tryAcquire是由子类实现，通过操作state进行判定当前是否允许当前线程获取执行权力，用来控制当前是否允许获取同步状态。true表示获取同步状态，不必加入同步队列中。如果返回了false，没有获取同步状态，则需要加入到同步队列中。继续往下执行：
#### addWaiter(Node mode)
首先将节点添加到等待队列中：
````
private Node addWaiter(Node mode) {
        // 构造一个Node，nextWaiter为null
        Node node = new Node(Thread.currentThread(), mode);
        // 获取到tail节点（也就是接下来，当前节点的前驱节点）
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // CAS尝试替换tail引用，如果成功，则返回
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 上述不成功，存在多线程竞争，则自旋
        enq(node);
        return node;
    }
````
#### enq(final Node node)
````
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 如果队列为空，先CAS设置一下head空节点，完事之后进行下一次循环
            if (t == null) { 
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 设置当前节点的prev，然后CAS设置设置tail，和前驱节点的next
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
````
添加队列成功之后，我们继续往下看，还是那张图
![](http://upload-images.jianshu.io/upload_images/1583231-48c676e2cb52cc0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

#### acquireQueued(final Node node, int arg)

acquireQueued主要是处理正在排队等待的线程。自旋、阻塞重试获取。如果获取成功则替换当前节点为链表头，然后返回。在获取过程中，忽略了中断，但将是否中断的返回了。
````
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 死循环自旋，不断尝试获取同步状态
            for (;;) {
                //获取当前节点的前驱节点
                final Node p = node.predecessor();
                // 只有前驱节点是head，也就是说排队排到当前借钱，才有可能获取同步状态
                // 如果允许获取同步状态，则将当前节点设置为head，设置其他标记，并返回，终止自旋
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 在上面同步获取失败后，有可能不是头节点的后继节点，这时没有资格获取同步状态，就需要休眠
                // 下面代码上面讲过，进一步检查和更新节点状态，判断当前节点是否需要park，减少占用CPU，等待前驱节点释放同步状态将它唤醒
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 如果失败，取消获取同步状态，移除节点，上文已讲
            if (failed)
                cancelAcquire(node);
        }
    }
````
#### selfInterrupt
获取锁过程中，忽略了中断，在这里处理中断
````
static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
````
获取分析完了，我们看一下，同步代码执行完毕，同步状态是如何释放的。
### 独占式释放同步状态
```
 public final boolean release(int arg) {
        //首先调用子类重写方法tryRelease，返回true标识标识允许释放同步状态
        if (tryRelease(arg)) {
            //如果允许释放，则当前head即为要释放的node，只需要唤醒后继node即可， unparkSuccessor上文讲过
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
到此，我们走完了独占式锁的获取与释放。简要概述一下步骤：
* 尝试获取锁，如果不能获取，添加进队列
* 队列中该node进行自旋排队，尝试获取同步状态
* 如果当前节点不是head的下个节点，休眠，等待唤醒
* 唤醒后，检查自身是否已被interrupted，继续尝试获取锁
* 获取后，执行同步代码，
* 执行完毕后，release锁，唤醒下个节点
# -----------------------------------------------------
### 共享式获取同步状态
上源码：
![](http://upload-images.jianshu.io/upload_images/1583231-dd403f31ecbd15b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)
首先还是调用子类实现的tryAcquireShared，查看是否允许获取同步状态。如果首次获取结果大于等于0.则完成获取 。如果小于0，则表示不允许获取同步状态，进入队列。
#### doAcquireShared(int arg)
死循环自旋尝试获取锁
````
 private void doAcquireShared(int arg) {
        // 构造Node，添加到队列中，模式为Node.SHARED，查看Node构造函数
        // 可以看到，当前Node的nextWaiter（不是next，详看上文）为一个空node对象
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 拿到前驱node
                final Node p = node.predecessor();
                // 前驱node是head才有可能获取锁
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {   // tryAcquireShared大于等于0，允许获取锁
                       // 获取成功，需要将当前节点设置为AQS队列中的第一个节点
                       // 这是AQS的规则，队列的头节点表示正在获取锁的节点
                       // 下面讲解
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        // 同独占式
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 不解释，见上文
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 不解释，见上文
            if (failed)
                cancelAcquire(node);
        }
    }
````
#### setHeadAndPropagate
这是
````
private void setHeadAndPropagate(Node node, int propagate) {
        // 取到head做缓存
        Node h = head; 
        //将当前节点设置为head
        setHead(node);
        // propagate是tryAcquireShared返回的值 ，可以理解为Semaphore，是否还允许其他并发
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            // 并检查当前节点的后继节点为空或者后继节点的nextWaiter是否为SHARED，表明后继节点需要共享传递
            if (s == null || s.isShared())
                doReleaseShared();  // 进行share传递， doReleaseShared
        }
    }
````
可以看到这里与独占式的做了相似的事情，都进行了设置head之后，区别是共享式获取同步状态又进行了share传递，传递给下一个nextWaiter属性同样为SHAREED的节点，我们看一下doReleaseShared方法

####  doReleaseShared

```
private void doReleaseShared() {
 /*
         * 即使在并发，多个线程在获取、释放的情况下，确保释放的传播性,
         * 如果当前节点标记为SIGNAL（表示后继节点需要唤醒，按理说应该在当前节点释放的时候唤醒，但是此处是共享模式，故立即唤醒），则通常尝试头节点的unparkSuccessor 动作。
         * 但是如果他不符合唤醒的条件，为了确保能正确release，那么则把head的waitState设置为为PROPAGATE
         * 此外，在执行该代码时，为了以防万一有新
         * 节点的加入，或者我们CAS修改失败，所以我们的更新需要在循环中，不断尝试。
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // 失败了就继续loop  
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // 失败了就继续loop  
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```


这里最重要的是要多线程环境中理解doReleaseShared，一个线程A执行doReleaseShared，然后unparkSuccessor，线程B唤醒执行，这时候被唤醒的线程B运行，重新请求获取同步状态，修改head节点，唤醒线程C，然后依次唤醒D、E、F……每个节点在自己唤醒的同时，也唤醒了后面的节点，设置为head，这样就达到了共享模式。


注意h == head，我们看到上面有注释说```Additionally, we must loop in case a new node is added while we are doing this.```为了避免在执行到这里的时候。如果有两个新的节点添加到队列中来，一个节点A唤醒B之后，B恰好setHead了，此时head是B节点。此时A之前获得的head并不是新的head了，故需要继续循环，以尽可能保证成功性。

>可以看到 独占式与共享式的差别就是共享的传递：
独占模式唤醒头节点，头节点释放之后，后继节点唤醒
共享模式唤醒全部节点。

### 共享式释放同步状态
源码不贴了，调用的是上述的doReleaseShared()

### 响应中断获取锁
acquireInterruptibly和acquire差不多，acquireSharedInterruptibly和acquireShared差不多，区别就是抛出了InterruptedException。

# -----------------------完毕-------------------------
下一篇继续撸Reentrantlock
本人能力有限，分析的不够的地方，还望多多指正。
