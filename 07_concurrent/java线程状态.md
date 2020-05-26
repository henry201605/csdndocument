一文搞懂线程世界级难题---线程状态到底6种还是5种！！！

## 背景

先来解答一个世界级难题：

java线程有多少种状态？

答案是6种。

那为什么有的地方说是5种呢，那这一定是将操作系统层面的线程状态搞混了。

下面我们就分别介绍一下java线程的6种状态以及操作系统层面的5种状态：

## 1、java线程状态

java线程有6种状态，我们先来一个官方的依据

```java
public class Thread implements Runnable {
  public enum State {
        /**
         * Thread state for a thread which has not yet started.
         * <p>
         *  尚未启动的线程的线程状态。
         * 
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         * <p>
         * 可运行线程的线程状态。处于可运行状态的线程正在Java虚拟机中执行,但它可能正在等待来自操作系统的其他资源,例如处理器。
         * 
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         * <p>
         *  线程阻塞等待监视器锁的线程状态。处于阻塞状态的线程在调用{@link Object#wait()Object.wait}后等待监视器锁进入同步块/方法或重新输入同步块/方法。
         * 
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         * <p>
         *  等待线程的线程状态。线程由于调用以下方法之一而处于等待状态：
         * <ul>
         *  <li> {@ link Object#wait()Object.wait}没有超时</li> <li> {@ link #join()Thread.join}没有超时</li> <li> {@ link LockSupport #park()LockSupport.park}
         *  </li>。
         * </ul>
         * 
         *  <p>处于等待状态的线程正在等待另一个线程执行特定操作。
         * 
         *  例如,在对象上调用<tt> Object.wait()</tt>的线程正在等待另一个线程调用<tt> Object.notify()</tt>或<tt> Object.notifyAll )</tt>
         * 。
         * 调用<tt> Thread.join()</tt>的线程正在等待指定的线程终止。
         * 
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         * <p>
         *  具有指定等待时间的等待线程的线程状态。由于调用指定正等待时间的以下方法之一,线程处于定时等待状态：
         * <ul>
         * <li> {@ link #sleep Thread.sleep} </li> <li> {@ link Object#wait(long)Object.wait} with timeout </li>
         *  <li> {@ link #join .join} with timeout </li> <li> {@ link LockSupport#parkNanos LockSupport.parkNanos}
         *  </li> <li> {@ link LockSupport#parkUntil LockSupport.parkUntil}。
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         * <p>
         *  终止线程的线程状态。线程已完成执行。
         * 
         */
        TERMINATED;
    }
}
```

简单来介绍一下6种状态（NEW、RUNNABLE、BLOCKED、WAITING、TIME_WAITING、TERMINATED） 

1)）NEW：初始状态，线程被构建，但是还没有调用 start 方法;

2）RUNNABLED：运行状态，JAVA 线程把操作系统中的就绪和运行两种状态统一称为“运行中” ；

3）BLOCKED：阻塞状态，表示线程进入等待状态,也就是线程因为某种原因放弃了 CPU 使用权，阻塞也分为几种情况 ：

* 等待阻塞：运行的线程执行` wait()` 方法，jvm 会把当前线程放入到等待队列 ；

* 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被其他线程锁占用了，那么` jvm `会把当前的线程
  放入到锁池中 ；
* 其他阻塞：运行的线程执行 `Thread.sleep` 或者 `.join` 方法，或者发出了 I/O 请求时，JVM 会把当前线程设置为阻塞状态，当 sleep 结束、join 线程终止、io 处理完毕则线程恢复；

4）WAITING：等待状态，没有超时时间，要被其他线程或者有其它的中断操作； 

5）TIME_WAITING：超时等待状态，超时以后自动返回；
6）TERMINATED：终止状态，表示当前线程执行完毕 。

## 2、线程状态间的转换

借一个图来描述：

![image-20200526103703738](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200526103703738.png)

具体的转换场景，图中描述的比较清楚，此处不再详细赘述。

注意：

1）sleep、join、yield时并不释放对象锁资源，在wait操作时会释放对象资源，wait在被`notify/notifyAll`唤醒时，重新去抢夺获取对象锁资源。

2）sleep可以在任何地方使用，而wait，notify，notifyAll只能在同步控制方法或者同步控制块中使用。

3）调用obj.wait()会立即释放锁，以便其他线程可以执行`notify()`，但是notify()不会立刻立刻释放sycronized(obj)中的对象锁，必须要等`notify()`所在线程执行完`sycronized(obj)`同步块中的所有代码才会释放这把锁。然后供等待的线程来抢夺对象锁。

## 3、java代码中查看线程状态

我们可以通过打印堆栈信息来查看线程的状态，下面举一个实例：

```java
public class ThreadState {

    public static void main(String[] args) {
        new Thread(()->{
            while(true){
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        },"thread01_status").start();  //阻塞状态

        new Thread(()->{
            while(true){
                synchronized (ThreadState.class){
                    try {
                        ThreadState.class.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        },"thread02_status").start(); //阻塞状态

        new Thread(new BlockedDemo(),"BLOCKED-DEMO-01").start();
        new Thread(new BlockedDemo(),"BLOCKED-DEMO-02").start();

    }
    static class BlockedDemo extends  Thread{
        @Override
        public void run() {
            synchronized (BlockedDemo.class){
                while(true){
                    try {
                        TimeUnit.SECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}

```

通过打印堆栈信息来查看：（我使用的是IDEA，也可以直接用cmd命令打开命令窗口查看，效果是一样的）

1）获取java进程的pid;

```shell
S:\workspace\study\concurrent\thread-demo>jps
23328 Launcher
5376 ThreadState
14700
19708 Jps

```

2）根据上一步骤获得的 pid，继续输入 `jstack pid`（`jstack`是 java 虚拟机自带的一种堆栈跟踪工具。`jstack `用于
打印出给定的 java 进程 ID 或 `core file` 或远程调试服务的 Java 堆栈信息） 

```shell
S:\workspace\study\concurrent\spring-boot-thread-demo>jstack 5376
2020-05-26 11:19:31
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.131-b11 mixed mode):

"DestroyJavaVM" #19 prio=5 os_prio=0 tid=0x0000000002e44000 nid=0x5788 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"thread05_status" #18 prio=5 os_prio=0 tid=0x000000001f703000 nid=0x5efc waiting for monitor entry [0x00000000202df000]
   java.lang.Thread.State: `BLOCKED (on object monitor)`
        at com.example.springbootthreaddemo.demo02.ThreadState$BlockedDemo.run(ThreadState.java:49)
        - waiting to lock <0x000000076c0dc148> (a java.lang.Class for com.example.springbootthreaddemo.demo02.ThreadState$BlockedDemo)
        at java.lang.Thread.run(Thread.java:748)

"thread04_status" #16 prio=5 os_prio=0 tid=0x000000001f701000 nid=0x52e0 waiting on condition [0x00000000201df000]
   java.lang.Thread.State:` TIMED_WAITING (sleeping)`
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at com.example.springbootthreaddemo.demo02.ThreadState$BlockedDemo.run(ThreadState.java:49)
        - locked <0x000000076c0dc148> (a java.lang.Class for com.example.springbootthreaddemo.demo02.ThreadState$BlockedDemo)
        at java.lang.Thread.run(Thread.java:748)

"thread03_status" #14 prio=5 os_prio=0 tid=0x000000001f715000 nid=0x4fc4 runnable [0x00000000200df000]
   java.lang.Thread.State: `RUNNABLE`
        at com.example.springbootthreaddemo.demo02.ThreadState.lambda$main$2(ThreadState.java:35)
        at com.example.springbootthreaddemo.demo02.ThreadState$$Lambda$3/935044096.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"thread02_status" #13 prio=5 os_prio=0 tid=0x000000001f714000 nid=0x4748 in Object.wait() [0x000000001ffdf000]
   java.lang.Thread.State:` WAITING (on object monitor)`
        at java.lang.Object.wait(Native Method)
        - waiting on <0x000000076bc73f08> (a java.lang.Class for com.example.springbootthreaddemo.demo02.ThreadState)
        at java.lang.Object.wait(Object.java:502)
        at com.example.springbootthreaddemo.demo02.ThreadState.lambda$main$1(ThreadState.java:26)
        - locked <0x000000076bc73f08> (a java.lang.Class for com.example.springbootthreaddemo.demo02.ThreadState)
        at com.example.springbootthreaddemo.demo02.ThreadState$$Lambda$2/443308702.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"thread01_status" #12 prio=5 os_prio=0 tid=0x000000001f711000 nid=0xb28 waiting on condition [0x000000001fedf000]
   java.lang.Thread.State: `TIMED_WAITING (sleeping)`
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at com.example.springbootthreaddemo.demo02.ThreadState.lambda$main$0(ThreadState.java:15)
        at com.example.springbootthreaddemo.demo02.ThreadState$$Lambda$1/205797316.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
```

从上面的信息中可以看到：

1）`thread01_status`：状态为`TIMED_WAITING`,该线程是执行了`sleep()`,所以处于线程休眠状态；

2）`thread02_status`：状态为`WAITING`,该线程是执行了`wait()`,所以处于线程等待状态；

3）`thread03_status`：状态为`RUNNABLE`,该线程是正常运行中，处于运行状态；

4）`thread04_status`：状态为`TIMED_WAITING`,该线程执行了同步代码块，它先获取到了`BlockedDemo`的类锁，执行了`sleep()`,所以处于线程等待状态；

5）`thread05_status`：因为线程`thread04_status`获取到的类锁没有释放，所以`thread05_status`在执行`synchronized(BlockedDemo.class)`时是阻塞的，所以为`BLOCKED`状态。

## 4、操作系统层面线程状态

​    很多人会把操作系统层面的线程状态与java线程状态混淆，所以导致有的文章中把java线程状态写成是5种，在此我们说清楚一个问题，java线程状态是6个，操作系统层面的线程状态是5种。如下图所示：

![image-20200526153809263](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200526153809263.png)

下面分别介绍一下这5种状态：

1）`new` ：一个新的线程被创建，等待该线程被调用执行；

2）`ready` ：表示线程已经被创建，正在等待系统调度分配CPU使用权，或者时间片已用完，此线程被强制暂停，等待下一个属于它的时间片到来。

3）`running` ：表示线程获得了CPU使用权，正在占用时间片，正在进行运算中；

4）`waiting` ：表示线程等待（或者说挂起），等待某一事件(如IO或另一个线程)执行完，让出CPU资源给其他线程使用；

5）`terminated` ：一个线程完成任务或者其他终止条件发生，该线程终止进入退出状态，退出状态释放该线程所分配的资源。

需要注意的是，操作系统中的线程除去new 和terminated 状态，一个线程真实存在的状态是ready 、running、waiting 。

## 结语

至此，我们就把java线程状态以及操作系统层面的线程状态说清了，是不是以后再也不会混淆了，希望能帮助到大家。