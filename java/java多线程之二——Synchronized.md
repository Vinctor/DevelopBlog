在java多线程并发编程中,```Synchronized```一直占有很重要的角色。```Synchronized```通过获取锁来实现同步。先来看一下,它的使用方法:
 ````
package com.Vinctor.Tst;

public class VinctorSyncDemo {

	public static synchronized void staticSyncMethod() {
		System.out.println("static synchronized");
	}

	public synchronized void normalSyncMethod() {
		System.out.println("normal  SyncMethod");
	}

	public void normalInnerMethod() {
		synchronized (this) {
			
		}
	}

	public void staticInnerMethod() {
		synchronized (VinctorSyncDemo.class) {

		}
	}
}

````
如上，分为三种方式:
* 如```staticSyncMethod```，修饰静态方法，锁是当前类的```类对象```，位于方法区
* 如```normalSyncMethod```，修饰普通实例方法，锁是当前```实例对象```，位于堆
* 修饰代码块，如```normalInnerMethod```与```staticInnerMethod```，其中```synchronized (this)```与```normalSyncMethod```效果相同，```synchronized (VinctorSyncDemo.class) ```与```staticSyncMethod```效果相同。

我们将上述代码```javap```之后，看一下字节码指令：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-c9e2d031ec7f6d52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

可以看到看到```staticInnerMetho```方法在执行到synchronized代码块时，有两个指令```monitorenter```和```monitorexit ```，而下面异常表指向的位置10，也同样执行了```monitorexit ```。可见同步代码块的实现是使用```monitorenter```和```monitorexit ```两个代码块进行获取锁已释放锁的，发生异常之后，也同样会释放锁。JVM 必须保证```monitorenter```有```monitorexit ```相对应。当监视器一个线程被持有时，那么这个线程就持有了锁，其他线程就不能获取这个监视器，直至```monitorexit ```释放锁。同步方法同样也可以用着两个指令获取和释放锁。

当一个线程试图访问同步方法或者同步代码块时，首先需要获取锁，退出同步方法或者同步代码块时需要释放锁。可以看出锁在```Synchronized```中起到至关重要的作用。锁是什么呢？他是怎么存储的呢？
通过以前的[文章](http://www.jianshu.com/p/860c259c8aad)，我们了解到实例对象存储在堆中，类对象存储在方法中，一个对象区域包括：对象头，对象数据，对其数据。此处对象头中存储着Synchronized的锁。对象头中有一个称为```Mark Word```的区域，存储着对象的 hashCode，分代年龄和锁标记位。如图：
                                                          
![image.png](http://upload-images.jianshu.io/upload_images/1583231-75f68b258640e11c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行期间，mark word 中存储的数据锁的标记位的变化而变化，其可能变化情况如下：

![image.png](http://upload-images.jianshu.io/upload_images/1583231-04e0f78947664265.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，分为很多锁：偏向锁，轻量级锁，重量级锁。

# 锁的分类

内置锁按照状态分为：无锁状态，偏向锁，轻量级锁，重量级锁。四中状态随着锁的竞争加剧而升级，但是不是回退降级。（本文仅讨论 HotSpot）

## 锁的升级

虚拟机开发人员研究发现，大多数情况下，多线程存在竞争的情况很少，为了避免同一线程多次进行锁的获取，故引入了偏向锁的概念。


当一个线程A访问同步代码块的时候，
首先检查锁标志位：如果为```01（无锁或偏向锁）```，再检查是否为偏向锁，
* 如果为```0（不是偏向锁）```，会通过 CAS 操作（下面将解释）在对象头中存储当前访问的线程 A 的ID ；
* 如果为```1（是偏向锁）```，检查一下对象头中是否是当前线程A 的 ID，如果是，则表示获取到锁，执行代码块。接下文，
接上文，如果不是当前线程A 的 ID，则尝试使用 CAS替换线程  ID，如果成功，则表示获取锁；如果失败，则表示其他线程正在持有锁，这时出现了竞争。我们假设当前线程 B 持有该锁。出现了竞争，我们这时需要撤销线程 B 的偏向锁（解锁）。
当线程 B 运行到安全点或者安全区域的时候，线程 B 暂停，这是检查线程 B是否已经退出了同步代码块，
* 如果线程 B已经退出了同步代码块，则解锁，将对象头 的线程 ID 清空，是否偏向锁标记位0（设为无锁状态），线程 A 继续通过 CAS 获取锁。
* 如果线程 B 没有退出同步代码块，则表示线程 A 不能获取锁，这时锁升级，升级为```轻量级锁```。


锁升级轻量级锁之后，对象头中不在存储线程 ID 等信息，而是将这些信息拷贝至持有锁的线程栈中锁记录中，再将对象头指向该地址。上面的例子🌰中，
* 线程 B 持有锁，在线程 B 的栈中分配锁记录，并将对象头数据拷贝进去，这时锁的标志位为```00（轻量级锁）```，并将对象头指针指向该线程 B 的栈，线程 B 唤醒，并继续执行代码；
* 此时线程 A 也是分配锁记录，并拷贝对象头中 Mark Word，但是线程 A 还是不能获取到锁。
这时线程 A还是进行 CAS 操作，企图将对象头指向自己的锁记录，
* 如果替换成功，则表示线程 A 获取到了锁，执行同步代码块；
* 如果还是不成功，这时线程 A 将执行```自旋```CAS（自旋：顾名思义，自己转着玩儿，也就是线程 A 不暂停等待，也不阻塞，而是执行一些无用的代码，此时空转，也占用 CPU 时间，可以想象一个while 循环，目的就是稍等一下，看看持有锁的线程是是否很快就释放锁），自旋的过程中尝试 CAS 替换对象头指针，当自旋一定数目之后，线程 A 还是没有获取到锁（够悲催的），这时锁升级，升级为重量级锁，此时标志位```10```。升级为重量级锁之后，线程 A 这是不在争抢资源，而是挂起当前线程，等待其他线程释放锁之后将它唤醒。

拥有轻量级锁的线程执行完同步代码块后，需要解锁轻量级锁，这是还是需要使用 CAS 操作，将栈中锁记录替换会对象头，如果成功，表示没有竞争；如果失败，表示存在竞争，其他线程尝试获取过锁，那就需要在释放锁的过程中，唤醒被挂起的线程。

贴一张收藏的流程图（出处不明，如侵权，请告知）：
![点击可放大，可查看原图](http://upload-images.jianshu.io/upload_images/1583231-bdab3b26c3d71904.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/2240)


# 一个🌰
以上就是锁升级的过程，比较乱，举个现实生活的栗子，上厕所。厕所几位锁（对象）。

为了方便对比，我们定一个规则，当一个人上厕所时，需要将自己的牌子挂在厕所的门上，上完厕所，从厕所出来就不必摘下牌子，下次再上的时候就不需要挂了，只是看一下这个牌子是不是自己的就可以了。此为偏向锁。

当你再上厕所的过程中，小明过来了也想上厕所，就看到了牌子，不是自己的，想要把牌子换成自己的，但是这是你还在上厕所，没办法，小明只能在厕所门前晃来晃去，等着你上完厕所出来。如果你能在短时间里出来，那小明就进去上厕所了。此时为轻量级锁。

但是万一你闹肚子，小明也不可能一直在外面等着，这是他就选择回座位等着，并告诉你一声：“哥们，上完告诉我一声！”，这时的小明不在主动去想要获取锁，而是等着你上完厕所出来喊他，他才上厕所。这是即为重量级锁。

# CAS
CAS，Compare and Swap即比较并替换，设计并发算法时常用到的一种技术。java 中的原子类以及concurrent包大量使用了该技术进行原子操作。
>CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做并返回false

JDK中有一个类Unsafe（sun.misc.Unsafe），它提供了硬件级别的原子操作。Unsafe类中的一个方法如下:
````
public final native boolean compareAndSwapInt(Object o, long offset,
                                              int expected,
                                              int x);
````
此为 native 方法，JVM 会将此方法映射为```cmpxchg```CPU 指令，该指令为原子操作，故多用于多线程环境中而不会产生数据错误。







                                                          








