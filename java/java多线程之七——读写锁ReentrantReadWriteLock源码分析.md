>本文基于java version "1.8.0_77"

在没有```ReentrantReadWriteLock```的时候,我们对资源进行读写操作时,为了确保正确的 读写,一般会使用```Synchronized```操作,如下:
````
  public synchronized void write(){
        //写操作
        notifyAll();
    }

    public synchronized void read(){
        //读操作  
        notifyAll();
    }
````
可以看到，读写操作都是互斥执行的。但这种写法存在一个问题，```读操作是可以并发进行的```，故这样互斥的写法存在计算机资源的浪费的问题。JUC中```ReentrantReadWriteLock```为了这一痛点应运而生了。
基本的规则就是：
* 读写互斥
* 写写互斥
* 读读可以并发

ReentrantReadWriteLock的特性：

* 公平性选择：支持非公平(默认)和公平的锁获取方式，吞吐量非公平优于公平。在读锁的获取过程中，为了防止写线程饥饿等待，如果同步队列中的第一个节点是写线程，则阻塞当前读线程。
* 重入：锁支持重进入。读线程在获取了读锁之后，能够再次获取读锁。而写线程在获取了写锁之后能够再次获取写锁。
* 锁降级：遵循```获取写锁——获取读锁——释放写锁```的次序，写锁能够降级成为读锁

# 基本用法
首先一个测试类：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-162d8bac12998de0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后进行调用：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-0cc53908d2c8b2b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后是输出结果：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-b9fce380b492f898.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，我用上面代码模拟了一个读写操作，并分别创建了3个现成对一个变量进行读写操作。
我们在demo结果中可以看到，
![image.png](http://upload-images.jianshu.io/upload_images/1583231-51635b7ab75ab3ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最开始的```Thread-0```和```Thread-1```读操作是可以并发执行的，

![image.png](http://upload-images.jianshu.io/upload_images/1583231-6dd61344183290be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来的三个写操作是互斥执行的。

下面我们从源码角度来分析一下```ReentrantReadWriteLock```。

# 源码分析

因为ReentrantReadWriteLock是基于AQS的，在阅读下面内容前，建议先阅读[java多线程之五——JUC核心AbstractQueuedSynchronizer(AQS)源码分析](http://www.jianshu.com/p/4a6d4ed88b1d)文章的内容。

ReentrantReadWriteLock实现了读写锁的接口 ReadWriteLock：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-c4b5874f8387fb71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 构造函数

![image.png](http://upload-images.jianshu.io/upload_images/1583231-d1363c254a3b48c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ReentrantReadWriteLock有两个构造函数，初始化时对是否使用公平锁进行设定，默认是非公平锁，并实例化了sync类。而Sync类继承了AQS类。
我们可以看到，ReentrantReadWriteLock为读写操作分别设置了一个锁```ReadLock```和```WriteLock```。两个锁的构造函数传入了当前ReentrantReadWriteLock类的实例，其实也只是用到了刚刚实例化了的sync。这两个锁也将是我们研究的重点。

读写锁```ReadLock```和```WriteLock```两个内部类的结构很简单，和之前分析过的[ReentrantLock源码](http://www.jianshu.com/p/417c8d7becf9)一样，如下图：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-621d1ed5c96dfe83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 状态标识

在ReentrantLock中我们用AQS中state变量标识线程的重入次数，而读写锁中需要标识两个状态：读状态与写状态。需要存储多个读状态与一个写状态。
那么如何使用一个int整形标识两个状态呢？
在android中，measure过程中，高两位用来测量控件的mode，其余的低位来测量控件的大小。
而在读写锁中同样使用了这一方法，读写锁是将变量切分成了两个部分，高16位表示读（共享锁），低16位表示写（独占锁）。如下图（图片来源于网络，如侵权望告知）：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-16fecb8dd01853e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过位运算进行计算，
假设当前同步状态值state为S：
假设S值为：```00000000000000110000000000000011```读写都位3
写状态等于 ```S & 0x0000FFFF```（将高16位全部抹去）,对应方法```Sync::exclusiveCount(int)```
````
进行&运算
00000000000000001111111111111111     EXCLUSIVE_MASK独占运算掩码
00000000000000110000000000000011     state值运算掩码
—————————————————————————————————    与运算
00000000000000000000000000000011     得到写状态的值
````
获取当前的读状态：等于``` S >>> 16```（无符号补0右移16位）。抹掉后16位值，（无符号右移，不管是正数还是负数，左边补零）,对应方法```Sync::sharedCount(int)```
````
进行无符号右移运算
00000000000000110000000000000011     state值
————————————————————————————————     S >>> 16
00000000000000000000000000000011     得到读状态的值
````
当写状态增加1时，由于写16位在低位，直接应该等于```S + 1```；

当读状态增加1时，应该先将1增至高16位```(1 << 16)```，然后再相加。等于```S + (1 << 16)```，也就是```S + 0x00010000```
````
1<<16:
00000000000000000000000000000001
————————————————————————————————     进行左移16位
00000000000000010000000000000000

然后相加：
00000000000000010000000000000000     1<<16之后的值
00000000000000110000000000000011     原state值
————————————————————————————————     相加
00000000000001000000000000000011     计算结果
````

## 写锁WriteLock
首先是构造函数：
````
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
````
获取到外部类ReentrantReadWriteLock的变量```sync```，进而进行下面的加锁解锁操作。读锁也是同样的操作，可以了解到，其实读锁与写锁中都持有同一个Sync，这样才能达到读写互斥的目的。

再继续看一下加锁操作：
````
        public void lock() {
            sync.acquire(1);
        }
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }
        public boolean tryLock( ) {
            return sync.tryWriteLock();
        }
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }
````
有没有很熟悉？这跟我们上篇讲到的[ReentrantLock](http://www.jianshu.com/p/417c8d7becf9)一个套路：
>调用AQS的```acquire```系列方法，然后AQS调用Sync实现的```tryAcquire```系列方法来来确定当前线程能否获取同步状态，如果可以获取，则执行同步代码；如果不允许获取，则进入由AQS管理的等待同步队列进行自旋等待（[AbstractQueuedSynchronizer(AQS)源码分析](http://www.jianshu.com/p/4a6d4ed88b1d)）。

注意此处调用的是```独占式的获取锁```，这是因为写操作与写操作，写操作与读操作都是互斥的。



执行流程1：WriteLock:lock()->Sync:tryAcquire(int acquires)

````
        protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {//标识当前已经有锁的获取操作
                // 如果c！=0，写锁=0，则表示读锁！=0，当前有读操作正在进行
                // 如果c！=0，当前已获得写锁的现成不是当前线程，则表示此此获取不是写锁重入，需要等待
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //如果已经等待的写锁加上当前即将获取的写锁超过65536，则超过最大统计值，抛出异常
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 设置state值，获取写锁 获取同步状态
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
````

writerShouldBlock()方法是由子类```NonfairSync```和```FairSync```类实现，这里体现在公平与否，与ReentrantLock类似
在公平锁中，
````
final boolean writerShouldBlock() {
    return hasQueuedPredecessors();
}

 public final boolean hasQueuedPredecessors() {
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
}
````
进行了hasQueuedPredecessors()判断，判断等待队列中是否还有比当前线程更早的, 如果为空，或者当前线程线程是等待队列的第一个时才占有锁。

在非公平锁中，
````
final boolean writerShouldBlock() {
    return false; // writers can always barge
}
````
直接返回了false，故非公平的情况下，写锁可以不必按照时序进行获取。


>可以看到在写锁获取的过程中，不仅要考虑重入的情况，还存在读写是否存在的情况，也就是读与写不能同时获取锁。只有等待其他线程都释放了读锁，写锁才能尝试获取。写锁一旦获取到，后续的读写锁都将阻塞进入等待队列。

写锁的释放：
````
        public void unlock() {
            sync.release(1);
        }

        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
````
与ReentrantLock相对比，很相似，只是简单的将state的低16位-1。当写状态为0的时候，表示写锁被完全释放。

## 读锁ReadLock
````
        public void lock() {
            sync.acquireShared(1);
        }
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
        }
        public boolean tryLock() {
            return sync.tryReadLock();
        }
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
        }
````

读锁的获取是共享式获取的，这时因为，读锁允许被多个线程同时获取，多个线程可以并发的进行读操作。
我们来看一下还是相同的套路：
````
        public void lock() {
            sync.acquireShared(1);
        }
      
        protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            //取低16位的值，也就是写锁状态位：不等于0表示写锁被占用
            //同时写锁被其他线程占用，获取读锁失败，返回小于0的数，表示获取失败
            //此处如果写锁是当前线程，也会可能获取到锁，因为存在写锁到读锁的降级，后面会讲
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            
            //获取高16位值，读锁的状态
            int r = sharedCount(c);
            //根据公平性不同，有不同的读锁获取策略，返回是否阻塞当前读锁获取操作。readerShouldBlock后面会详细说明
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //CAS修改高16位的读锁状态，成功获取到读锁
                //以下设置计数
                if (r == 0) {  
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                //成功，返回1
                return 1;
            }
            //如果上述的CAS更改state值失败，则执行fullTryAcquireShared，自旋重试获取，下面将分析
            return fullTryAcquireShared(current);
        }
````
接下来看一下其中的几个重点方法：

#### readerShouldBlock()
和writerShouldBlock一样，在公平、非公平中有不同的实现：
公平锁中：
````
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
`````
判断是否有前置节点，不解释了
看一下非公平锁中，
````
    final boolean readerShouldBlock() {
          return apparentlyFirstQueuedIsExclusive();
    }
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }

````
可以看到最终执行了AQS的```apparentlyFirstQueuedIsExclusive```方法， 如果为了防止写线程饥饿等待，如果同步队列中的第一个线程是以独占模式获取锁（写锁），那么当前获取读锁的线程需要阻塞，让队列中的第一个线程先执行。

#### firstReader  HoldCounter 等
先看上面的代码，我截取下来：
````
             int r = sharedCount(c);
              if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
````
前面提到了，读锁是共享式获取锁的，多个读线程可以并发进行读取数据，获取一个读锁+1，释放一个读锁-1.其中HoldCounter 是用来记录线程获取读锁（重入）的次数。HoldCounter 类代码如下：

````
        /**
         * A counter for per-thread read hold counts.
         * Maintained as a ThreadLocal; cached in cachedHoldCounter
         */
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }
````

很简单：1个用来记录重入次数的int变量与一个记录线程ID的long变量。
在CAS设置State值设置成功之后，
* 我们看到如果是首个线程获取锁```r == 0  ```，则表示当前线程是首个获取锁的线程，则不必存储到HoldCounter ，直接用两个变量记录一下即可（firstReader 与 firstReaderHoldCount ）；
* 如果不是已经有线程获取到了锁，且```firstReader == current```,表示已经获取锁的线程是当前线程，则该次获取锁为第一个读锁线程重入  ，直接```firstReaderHoldCount```即可；
* 如果当前线程不是首次获取锁的线程，且也不是重入，即当前线程非第一个读锁线程，那么就需要使用一种数据结构来存储标记```哪个线程获取了几次锁```。这个时候```ThreadLocal```就派上了用场。看这种情况下的代码：
````
                  HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;            
````
````
          private transient ThreadLocalHoldCounter readHolds;

          static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
                  public HoldCounter initialValue() {
                        return new HoldCounter();
                  }
          }
````
里面有一个readHolds全局变量，这个是ThreadLocalHoldCounter 类，ThreadLocalHoldCounter 类继承自ThreadLocal。由此可以看到前面讲过的```哪个线程获取了几次锁```是由```ThreadLocal```保存的。```ThreadLocal```将ThreadLocalHoldCounter 对象绑定到特定的线程上。ThreadLocalHoldCounter 在Sync无参构造函数中进行初始化的。

了解了```ThreadLocalHoldCounter ```，我们看一下上面的流程，首先拿到已经缓存过的cachedHoldCounter，这个cachedHoldCounter是上次读锁获取过程中使用赋值的，然后判断cachedHoldCounter的线程id是否是当前线程的id或者cachedHoldCounter为空，
* 如果不是，则```cachedHoldCounter = rh = readHolds.get()```，将当前线程对应的HoldCounter从ThreadLocal中取出来；
* 如果cachedHoldCounter 不是空且cachedHoldCounter 的线程ID为当前线程ID，且HoldCounter的读锁获取此时为0，则加入到readHolds中  。

总的来说，上述就是为了获取到当前线程对应的HoldCounter。此后再进行```rh.count++```。

#### 读锁的释放
````
        public void unlock() {
            sync.releaseShared(1);
        }
````
````
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                //如果firstReaderHoldCount 为1，那就当前线程释放锁
                //如果不是1，则--
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                //拿到当前线程的HoldCounter ，然后-1或者从ThreadLocalHoldCounter 释放出来。
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            //自旋修改state值
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
````
释放过程中做了3件事，
* 若为第一次读锁线程，设置firstReader 与firstReaderHoldCount ；
* 若不为第一次读锁线程，则获取HoldCounter 并修改计数；
* 最后CAS修改State值


# End
总结一下：
#### State
读写锁中State表示了两种状态，写状态占低16位，读状态占高16位，两者通过位运算进行操作。
#### 写锁
独占式获取锁，获取过程与ReentrantLock类似，当前没有读写操作才会获取锁，
#### 读锁
共享式获取锁，如果存在非当前线程获取了写锁，则进入等待队列；否则，获取锁，并使用了ThreadLocalHoldCounter（ThreadLocal）存储当前线程与HoldCounter的关系。


























