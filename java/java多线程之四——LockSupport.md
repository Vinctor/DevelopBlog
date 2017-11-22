>本文基于java version "1.8.0_77"

LockSupport（java.util.concurrent.locks.LockSupport）是Java中底层类，提供了基本的线程同步原语。JUC中同步框架核心AQS（AbstractQueuedSynchronizer），就是通过使用LockSupport来实现线程的阻塞与唤醒的。我们先了解一下LockSupport类，为了解AQS做准备。
LockSupport中的两个核心方法：
* public static void park()
*  public static void unpark(Thread thread)

```park```译为“停车”，官方文档意为：许可。为了方便理解，在这里我们可以理解为```阻塞，等待，挂起```，而```unpark```我们理解为```唤醒，恢复```。
这些字眼是不是很熟悉？我们在使用多线程的时候会调用```object.wait()```和```object.notify()，object.notifyall()```来达到等待和唤醒的功能。此处我们可以比较着来学习。

与object的```先wait，后notify```方法不同的是，park与unpark无需担心调用时序问题，可以```先park，后unpark```，也可以```先park，后park```。还有一点需要注意，多次连续给一个线程下发许可，但是这中间并没有消耗的情况下，只会保留一个许可。（可以理解为许可的只有```有```或```没有```之分，而没有数量的多少）
 当线程A调用```park()```后，会申请一个许可证：
* 如果没有许可，会阻塞当前线程，直至有其他线程下发许可（LockSupport.unpark(线程A)）。
* 如果此时已经能够有一个许可，则可以继续往下执行代码。


我们来看下面的例子：
````
 public static void main(String[] args) {
        LockSupport.park();// 获取许可
        System.out.println("END");
    }
````
运行上述代码，会发现，该代码不会打印END。因为当前线程申请一个许可而没有线程给他许可，故一直阻塞，进程不会关闭。

多线程中通信的例子：
![吃鸡.png](http://upload-images.jianshu.io/upload_images/1583231-7cd2306d600d00aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)
结果：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-055c392fc4ade6a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

上图中可以看到，boyThread通过park阻塞线程，直至girlThread调用unpark唤醒。

看一个特殊例子：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-8fa78f7a2dbef6c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


结果：
![image.png](http://upload-images.jianshu.io/upload_images/1583231-1405badfb85ce9f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，即便是已经下发了4个许可，但是实际上只有一个许可，也就是说许可存在的个数只有0或1。

其他方法：
*  ```void park(Object blocker)```阻塞线程时添加附加信息，用来记录线程是被谁堵塞的，程序出现问题时候，通过线程监控分析工具可以找出问题所。
*  ```void parkNanos(long nanos) ```参数为阻塞超时时间，超时时间过后，如果还没有下发许可，则自动唤醒
* ```void parkNanos(Object blocker, long nanos) ```同上，方法重载
* ```void parkUntil(long deadline)```参数为阻塞截止时间，为绝对时间，到达时间后，如果还没有下发许可，则自动唤醒
* ```void parkUntil(Object blocker, long deadline)```同上，方法重载

## 与Object中wait和notify的区别
* 比wait/notify更加轻便灵活。LockSupport直接操作线程，而wait/notify则是需要一个Object对线程进行操作。
* 不依赖监视器（锁）。wait/notify需要在synchronized中进行调用，它首先需要获取到锁，才能进行下面的操作。而LockSupport则是直接进行调用。（synchronized与wait/notify配合使用，ReentrantLock与Condition配合使用。而Condition就是使用LockSupport实现等待与唤醒的）

End











