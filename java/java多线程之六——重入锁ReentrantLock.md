>本文基于java version "1.8.0_77"

ReentrantLock（java.util.concurrent.locks）（译为：重入锁）是java 5.0之后新加入的并发机制。他的出现并不是替代之前的内置锁[synchronized](http://www.jianshu.com/p/81e6f64c0fec)，而是在内置锁不适用时，可以提供更加灵活的加锁机制。

# ReentrantLock使用：
````
Lock lock = new ReentrantLock();  //可选参数：boolean 是否公平

// lockInterruptibly()，可中断的获取锁
// tryLock()//如果已经被lock，则立即返回false不会等待，达到忽略操作的效果   
// tryLock(long timeout, TimeUnit unit) //如果已经被lock，尝试等待，看是否可以获得锁，如果超市仍然无法获得锁则返回false继续执行  

lock.lock(); / /如果被其它资源锁定，会在此等待锁释放，达到暂停的效果  
try {   
    //do something
}  
finally {  
    lock.unlock();   // 不要漏掉
}  
````
ReentrantLock的使用很简单，首尾加解锁，中间是执行同步代码。ReentrantLock同样是一种互斥，可重入锁，同时它支持了获取锁的时候的公平性与否。
* 互斥：表示每次只有一个线程可以获取锁执行同步代码；
* 可重入：表示同一个线程可以对同一个锁的重复获取，比如在循环中加解锁。
* 公平性：等待中的线程是否按照时间顺序获取锁。


### ReentrantLock与synchronized

ReentrantLock与synchronized相同，都是可重入锁，互斥锁。在提供了与synchronized相同的锁功能的同时，ReentrantLock还提供了如下功能：
* 锁等待可以设置超时时间 tryLock(long, TimeUnit)，如果取得成功则继续处理，取得不成功，可以等下次运行的时候处理，所以不容易产生死锁。而synchronized则一旦进入锁请求要么成功，要么一直阻塞，所以更容易产生死锁。此方法仅在调用时锁为空闲状态才获取该锁。如果锁可用，则获取锁，并立即返回值true。如果锁不可用，则此方法将立即返回值false。
* 可以设置可中断的锁等待 lockInterruptibly()
* 可以设置锁等待的公平性  ReentrantLock(boolean fair)

需要注意的地方：lock 必须在 finally 块中释放。否则，如果受保护的代码将抛出异常，锁就有可能永远得不到释放！

# Lock

ReentrantLock实现了Lock接口，Lock接口定义了如下方法：

* ````void lock()````  获取锁，可等待
* ````void lockInterruptibly() throws InterruptedException````可中断的获取锁
* ````boolean tryLock();````如果已经被lock，则立即返回false不会等待，达到忽略操作的效果  
* ````boolean tryLock(long, TimeUnit) throws InterruptedException;````如果已经被lock，尝试等待，看是否可以获得锁，如果超时仍然无法获得锁则返回false继续执行  
* ````void unlock();````解锁
* ````Condition newCondition()```` 译为：“条件”，创建一个新的Condition，绑定到当前Lock上。Lock 替代了 synchronized 方法和语句的使用，而Condition 替代了 Object 监视器方法的使用。在Condition中，await()相当于wait()，signal()相当于notify()，signalAll()相当于notifyAll()。

# Fuck源码

我们从ReentrantLock调用的角度来看一下源码：

看一下ReentrantLock的类结构

![image.png](http://upload-images.jianshu.io/upload_images/1583231-98c45b580455f88a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

我们翻看ReentrantLock的源码可以看到，ReentrantLock看起来更像是一个代理类，他的所有对外公布的方法全部都是有两个内部类来实现的：FairSync（公平锁）和NonfairSync（不公平锁），对于是否为公平锁，使用了策略模式，针对公平与否，使用不同的获取锁的策略。而这两个类继承了同一个类：Sync。而Sync类继承了AbstractQueuedSynchronizer类。

我们在上篇的[AQS源码分析](http://www.jianshu.com/p/4a6d4ed88b1d)中讲到，如果线程获取锁失败，则进入等待队列（FIFO），进入队列是按照时间顺序的。

AQS中有两个重要部分：
1. 等待队列：是为了管理等待的线程；
2. ```state```系方法如下图，由子类调用，用来控制是否允许获取锁。例如：ReentrantLock中用它来表示所有线程呢个已经重复获取该锁的次数，Semaphore用它来表示剩余的许可数量，FutureTask用它来表示任务的状态（未开始，正在运行，已结束，已取消等）。这样子类不用关心等待队列如何工作，只需要控制state就可以实现一个lock。

![image.png](http://upload-images.jianshu.io/upload_images/1583231-50c35ffd0db54cda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

### 非公平锁&重入
NonfairSync：tryAcquire(int acquires)

![image.png](http://upload-images.jianshu.io/upload_images/1583231-0491f1d392f96d4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到非公平锁的tryAcquire方法调用了nonfairTryAcquire方法，我们继续往下看：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-ddd6b83657d988ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

假设一个线程A尝试获取锁，首先判断当前state的值，
* 如果为0，则表示当前并没有线程获取当前锁，那就尝试将state设置为1，如果失败，则表示已经有线程获取了锁，线程A获取失败，返回false；如果成功，表示当前线程A获取了锁，并保存线程A，设置标记一下当前独占模式的线程。
* 如果不为0，表示已经有线程获取过锁。这时有两种可能：有可能是线程A已经获取过锁，此时是线程A的重入；也有可能是其他线程（反正不是线程A）获取锁。这是需要先判断一下是否是重入（```if (current == getExclusiveOwnerThread()) ```），如果是重入情况下，则继续在state的值的基础上+1，这时重入，再次获取了锁。


### 公平锁&重入

首先，看公平锁的lock源码：
FairSync:tryAcquire()

![image.png](http://upload-images.jianshu.io/upload_images/1583231-0bd62f457e6eefd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

可以看到，与上面讲到的非公平锁极其类似，唯一不同的是，在首次获取锁的时候，进行了```hasQueuedPredecessors()```判断，判断等待队列中是否还有比当前线程更早的, 如果为空，或者当前线程线程是等待队列的第一个时才占有锁。我们看一下它的源码：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-45029fff5d378f66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

这是对同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。
这里判断了```(s = h.next) == null ```是为了，防止此时有另外的线程在同时抢占锁，并获取锁，故如果为null，则进入等待队列。

## End

