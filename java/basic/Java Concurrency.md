# The Art Of Java Concurrency Programming

## 第一章 前言

​		并发编程目的是让程序运行的更快，但是并发编程的速度面临很多的挑战，比如上下文切换，死锁等问题。

### 1.1	上下文切换

​		CPU通过给每个线程分配CPU时间片来实现多线程这个机制，CPU分配给每个线程的时间非常的短，所以要不停切换线程执行，让我们感觉是多个线程在执行。

​		当前任务执行一个时间片后会切换到下一个任务。但是，切换前会保留上一个任务的状态，便于切回来。所以任务从保存到再次加载的过程就是一次上下文切换。

​		上下文切换会影响多线程的执行速度。

#### 1.1.1	多线程一定快吗

​		当并发操作累加不超过百万次的时候，速度比串行要慢。

#### 1.1.2	如何减少上下文切换

​		□	无锁并发编程。多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使 			  用锁，如：将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据。

​		□	CAS算法。Java的Atomic包使用CAS算法来更新数据，不需要加锁

​		□	使用最少线程。

​		□	协程。在单线程里完成多任务的调度，并在单线程里维持多个任务间的切换。

### 1.2	死锁

​		说起死锁，先说说活锁和饥饿

​		饥饿是指某一个或者多个线程因为种种原因无法获得所需要的资源，导致一直无法执行，比如它的线程优先级可能太低，而高优先级的线程不断抢占它的资源，导致低优先级线程无法工作 

​		活锁，几个线程之间互相谦让，主动让出资源，所以没有一个线程可以同时拿到所有资源		

```java
public class DeadLockDemo{
    private static String A="A";
    private static String B="B";
    
    public static void main(String []args){
        new DeadLockDemo().deadLock;
    }
    
    private void deadLock(){
        Thread t1 = new Thread(new Runnable(){
           public void run(){
               synchronized(A){
                   try{
                       Thread.currentThread().sleep(1000);
                   }catch(Exception e){
                       e.
                   }
                   synchronized(B){
                   		syso("1");     
                   }
               }
           } 
        });
         Thread t2 = new Thread(new Runnable(){
           public void run(){
               synchronized(B){
                   synchronized(A){
                   		syso("2");     
                   }
               }
           } 
        });
    }
}
```

​		上述情况，在t1拿到锁之后，因为一些状况没有释放掉。

​		解决办法：

​		□	避免一个线程同时获取多个锁

​		□	避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源

​		□	尝试使用定时锁，使用lock.tryLock(timeout)来代替使用内部锁机制

​		□	对于数据库锁，加锁和解锁必须在一个数据库连接里，不然会出现解锁失败的情况。	

## 第二章 volatile关键字

Java并发的机制依赖于JVM的实现和CPU指令。

### 2.1 volatile的应用

​		多线程并发编程中synchronized和volatile都十分重要，volatile是轻量级的synchronized，它在开发中保证了共享变量的“可见性”。可见性的意思是当一个线程改变一个共享变量时，另一个线程能读到这个修改的值。它不会引起上下文切换和调度。

​		**1.volatile的定义与实现原理**

​		Java语言提供volatile，在某些情况下比锁要方便的多。如果一个字段被声明成volatile，Java在线程内存模型确保所有线程看到这个变量是一致的。 

​		在JMM内存模型中，每个线程都有一个工作内存，存储着共享变量的副本，线程无法操作其他线程的工作内存，这就有了可见性问题，volatile如何实现可见性呢，当CPU写数据的时候，有volatile声明的关键字进行写操作时，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在的缓存行写回内存，但是写回内存，如果其他处理器的缓存的值是旧的，那读出来操作也会有问题，这里会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己的缓存是不是过期了，当处理器发现自己的缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态。当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

## 第三章 多线程下的内存模型

### 3.1	Java内存模型

#### 3.1.1	并发编程模型的两个问题

​		并发编程中有两个问题：线程间如何通信及线程之间如何同步。通信是指线程之间以何种机制来交换信息。线程之间通信的机制有两种：共享内存和消息传递。

​		在共享内存的并发模型里，线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信。在消息传递的并发模型里，线程之间没有公共状态，线程之间必须通过发送消息来显示进行通信。

​		同步是指程序中用于控制不同线程间的操作发生相对顺序的机制。在共享内存并发模型中，同步是显示进行。在消息传递的并发模型里，由于消息的发送必须在消息的接收之前，因此同步是隐式进行的。

​		Java的并发采用的是共享内存模型，Java线程之间的通信是隐式进行的，整个通信过程对程序猿完全透明。

#### 3.1.2	Java内存模型（JMM）的抽象结构

​		在Java中，堆内存在线程之间共享。

​		Java线程之间的通信有JMM控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。抽象角度看，JMM定义了线程和主存之间的抽象关系：线程之间的共享变量存储在主内存中，每一个线程都有一个私有的本地，本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM一个抽象概念，并不真实存在。

​		![1563186217342](/asset/JMM.png)

​		如果线程A与线程B之间要通信的话，必须经历两个步骤。

​		1）线程A把本地内存A中更新过的共享变量刷新到主内存中去。

​		2）线程B到主内存中去读取线程A之前已经更新过的共享变量。			

​		假设如下情况：

​		初始时，三个内存中都存有变量x且初始化为0。在线程A执行时，把更新后的x值临时存放在自己的本地内存A中。当线程A与线程B需要通信时，线程A首先会把自己在本地内存中修改后的x的值刷新到主内存中去，此时主内存中的x变为了1。随后，线程B到主内存中去读取线程A更新的x的值，此时线程B的本地内存中的x的值变为了1。

#### 3.1.3	从源代码到指令序列的重排序

​		在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。分三种类型。

​		1）编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

​		2）指令级并行的重排序。

​		3）内存系统的重排序。

#### 3.1.4	并发编程模型分类

​		

### 3.2	重排序

​		重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。

#### 3.2.1	数据依赖性

​		如果两个操作访问同一个变量，且这两个操作有一个为写操作，此时这两个操作之间就存在数据依赖性。

| 名称   |          | 说明                         |
| ------ | -------- | ---------------------------- |
| 写后读 | a=1;b=a; | 写一个变量之后，再读这个位置 |
| 写后写 | a=1;a=2; | 写一个变量之后，再写这个变量 |
| 读后写 | a=b;b=1; | 读后写                       |

编译器和处理器在重排序时，会遵守数据依赖性，不会改变存在数据依赖关系的两个操作的执行顺序。

#### 3.2.2    as-if-serial语义

​     

#### 3.2.3	程序顺序规则

​	 

#### 3.2.4	重排序对多线程的影响

重排序是否会改变多线程程序的执行结果。

```java
public class ThreadDemo{
    int a = 0;
    boolean flag = false;
    
    public void writer(){//一个线程执行writer时，由于a与flag是没有联系的，所以会进行重排序
        a = 1;
        flag = true;
    }
    
    public void reader(){
        if(flag){
            int i = a*a;
		}
    }
}
```

### 3.3	顺序一致性

#### 3.3.1	

## 第四章 Java并发编程基础

### 4.1	线程简介

#### 4.1.1	什么是线程

​		一个Java程序的运行不仅仅是main()方法的运行，而是main()线程和多个其他线程的同时运行。

#### 4.1.2	为什么要使用多线程

​		更多的处理器核心

​		更快的响应时间

​		更好的编程模型

#### 4.1.3	线程优先级

​		现代操作系统基本采用时分的形式调度运行的程序，操作系统会分出一个个时间片，线程会分到若干个时间片，当线程的时间片用完了就会发生线程的调度，并等待着下次的分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而线程优先级就是决定线程需要多或者少分配一些处理器资源的线程属性。

​		在Java线程中，通过一个整型变量priority来控制优先级，优先级的范围充1~10，在线程构建的时候可以通过setPriority(int)方法来修改优先级，默认优先级是5，优先级高的线程分配时间片的数量要多于优先级低的线程。设置线程优先级时，针对频繁阻塞（休眠或者I/O操作）的线程需要设置较高的优先级。

​		**程序的正确性不能依赖线程的优先级高低**

#### 4.1.4	线程的状态

​		

### 4.2	启动和终止线程

#### 4.2.1	构造线程

#### 4.2.2	启动线程

​		调用start()方法。

#### 4.2.3	理解中断

​		它表示一个运行中的线程是否被其他线程进行了中断操作。中断好比其他线程对该线程打了个招呼，其他线程通过调用该线程的interrupt()方法对其进行中断操作。

​		线程通过检查自身是否被中断来进行响应，线程通过isInterrupt()来进行判断是否被中断，也可以调用静态方法Thread.interrupted()对当前线程的中断标识进行复位。如果该线程已经处于终结状态，即使该线程被中断过，在调用该对象的isInterrupted()方法将会返回false。

​		许多声明抛出InterruptedException的方法，这些方法在抛出这个异常之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出该异常，此时isInterrupt()方法将会返回false。

#### 4.2.5	安全终止线程

​		中断操作是一种简便的线程间的交互方式，这种交互方式最适合用来取消或终止任务。除了中断以外，还可以利用一个boolean变量来控制。

```java
//中断方式
public class ThreadInterrupt {
    private static class Worker extends Thread{
        @Override
        public void run() {
            while (true){
                if(Thread.interrupted()){
                    break;
                }
                //do things
            }
        }
    }

    public static void main(String[] args) {
        Worker w = new Worker();
        w.start();

        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        w.interrupt();
    }
}
```

```java
//boolean方式


```

```java
//暴力停止，当你需要
```

### 4.3	线程间的通信

#### 4.3.1	volatile

​		Java支持多个线程同时访问一个对象或者对象的成员变量，由于每个线程可以拥有这个变量的拷贝，所以程序在执行过程中，一个线程看到的变量不一定是最新的。

​		关键字volatile可以来修饰字段（成员变量），就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。

​		举个例子，有一个决定程序是否运行的变量boolean on = true，那么另一个线程可能对它执行关闭动作on=false，这里涉及多个线程对变量的访问，因此需要将其定义为volatile Boolean on = true，这样其他线程对它进行改变时，可以让所有感知到变化，多使用volatile会降低效率。

​		关键字synchronized可以修饰方法或者同步块的形式来进行使用，它主要是确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性与排他性。

​		任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块或者同步方法中，而没有获取到监视器的线程将会被阻塞在同步块和同步方法的入口处，线程进入同步队列，进入BLOCKED状态。等待获取了监视器的线程释放监视器。

#### 4.3.2	等待/通知机制

​		一个线程修改了一个对象的值，而另一个线程感知到了变化，然后进行相应的操作，整个过程开始于线程，而最后的执行结果又是另一个线程。前者是生产者，后者是消费者。

​		简单的办法就是让消费者线程不断地循环检查变量时候符合预期，比如下面的代码

```java
while(value != desire){
	Thread.sleep(1000);
}
doSomething();
```

如果条件满足退出while循环，从而完成消费者的工作。

| 方法名称         | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| notify()         | 通知一个在对象上等待的线程，使其从wait()方法返回，而返回的前提是该线程获取到了对象的锁 |
| notifyAll()      | 通知所有等待在该对象上的线程                                 |
| wait()           | 使线程进入WAITING状态，只有等待另外线程通知或被中断才会返回，调用wait后，会释放对象的锁，sleep不会释放锁 |
| wait(long)       | 超时等待一段时间，参数单位是毫秒                             |
| wait(long， int) |                                                              |

​		等待/唤醒机制，是指一个线程A调用对象O的wait方法进入等待状态，而另一个线程B调用了对象O的notify或者notifyAll方法，线程A收到通知后从对象O的wait方法返回，进而执行后续的操作。

​		1.使用wait，notify和notifyAll方法时需要先对调用对象加锁。

​		2.调用wait方法后，线程状态有RUNNING变为WAITING，并将当前线程放置到对象的等待队列。

​		3.notify或notifyAll方法调用后，等待线程依旧不会从wait返回，需要调用notify或 notifyAll释放锁后，等待			线程才有机会从wait方法返回		

​		4.notify方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll方法则是将等待队列中			的所有此线程全部移动到同步队列，被移动的线程状态由WAITING变为BLOCKED。

​		5.从wait()方法返回的前提是获得了调用对象的锁。

#### 4.3.3	等待/通知的经典范式

#### 4.3.4	管道输入/输出流

#### 4.3.5	Thread.join()

​		如果一个线程A执行了thread.join方法，其含义是：当前线程A等待thread线程终止之后才会从thread.join返回。

​		在下面的程序中，当你不加join时，主线程会迅速执行完，并输出数据已经保存，但是这是不符合逻辑的，应该先保存，再提示保存完毕，加入join后，三个非守护线程会根据睡眠时间执行结束，最后main线程结束，因为join的作用是让当前线程执行结束后，才可以接着往下执行，较快的线程必须等候

```java
public class ThreadJoin {

    public static void main(String[] args) throws InterruptedException {
        long starttime = System.currentTimeMillis();

        Thread t1 = new Thread(new DataOpertion("M1", 10_000L));
        Thread t2 = new Thread(new DataOpertion("M2", 20_000L));
        Thread t3 = new Thread(new DataOpertion("M3", 15_000L));

        t1.start();
        t2.start();
        t3.start();

        t1.join();
        t2.join();
        t3.join();

        long endtime = System.currentTimeMillis();
        System.out.printf("save data time is:%s, endtime is:%s",starttime,endtime);
    }
}
class DataOpertion implements Runnable{
    private String machinename;

    private Long spendtime;

    public DataOpertion(String machinename, Long spendtime){
        this.machinename = machinename;
        this.spendtime = spendtime;
    }

    public String getMachinename() {
        return machinename + "finsh.";
    }

    @Override
    public void run() {
        try {
            Thread.sleep(spendtime);//do some machine Opeations
            System.out.println(machinename + " completed data successfully.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 4.4 Sychronized

synchronized的加锁是给对象加锁，synchronized用的锁就是存在Java对象头里，HotSpot 对头主要包括两部分数据：Mark Word，Klass Pointer。

通过java的反汇编代码我们发现，在synchronized关键字的代码处有一处monitorenter指令，结束的时候有monitorexit指令，synchronized关键字的参数可以是一个任意对象，加锁其实并不是把对象锁住，而是把与它关联的monitor锁住。

monitorenter：每一个对象都会有一个监视器monitor与之关联，监视器被锁住，其他线程无法获取该对象的监视器。monitor中有两个非常重要的参数：owner，拥有这把锁的线程；recursions会记录线程拥有锁的次数，当一个线程拥有monitor后其他线程只能等待

monitorexit：执行monitorexit就是把计数器-1，减到0，代表释放了这把锁，**这里要注意，当同步代码块中的代码遇到异常时，会直接执行monitorexit，就是会释放锁**

同步方法：会隐式调用monitorenter和monitorexit。

![synchronized源码](/asset/synchronized源码.png)

**JVM源码分析synchronized：**

参加竞争时，其他的线程会被放在_cxq中，如果调用了wait()，会被放在等待队列中，如果释放了锁，但是重入了，这时候其他的线程会被放到 _EntryList中等待

![](/asset/monitor对象.png)

**执行synchronized代码块中的流程大致如下：**

1. CAS尝试把上面图中的owner字段设置为当前线程

2. 如果设置之前的owner指向当前线程，说明当前线程重入，执行recursions++，记录重入次数。
3. 如果当前线程是第一次进入monitor，设置recursions=1，_owner为当前线程，成功返回
4. 获取锁失败则等待锁的释放。

虚拟机在JDK1.6时对synchronized做了一定的优化，见第12.2章。

**Java的对象布局：**

在JVM中，对象在内存中分为三块区域，对象头，实例数据和对齐填充

![](/asset/对象头.png)

**对象头**

对象头由Mark Word和Klass pointer，Mark Word用于存储自身运行时的数据，另一部分是类型指针，及对象指向它的类元数据的指针

**Mark Word**

Mark Word存储的数据，如哈希码，GC分代年龄，锁状态的标志，线程持有的锁，偏向锁ID，偏向时间戳

**32位虚拟机的Mark Word：**



**64位虚拟机的Mark Word：**

Mark Word占64位，存储结构如下：

![](/asset/64位Mark Word.png)

## 第5章 Java并发编程的原子性，可见性，有序性

1.原子性：一个操作，要么都成功，要么都失败

2.可见性：可见性是指当一个线程修改了共享变量后，其他线程能够立即得知这个修改。

3.有序性：

### 	5.1 原子性

说原子性之前，我们先进入一段代码

```java
public class AtomicTest {

    public static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                    count++;
                    System.out.println(Thread.currentThread().getName()+"---->"+count);
            }
        });

        Thread t2 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                    count++;
                    System.out.println(Thread.currentThread().getName()+"---->"+count);
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

我们运行上面的代码时，会发现两个线程打印出来的count值有可能是一样的，最终的count是小于200的，是的，count++，并不是一个原子操作，根据定义，说明count++是可以被打断的，为什么呢，count++分为三步

①获取count的值②count+1③将新值赋值给count，这是计算机内部操作count++，虽然我们只写一步。

那么问题就来了，线程A刚执行了①（假设count=0），执行权切换到线程B并且完成了+1（count=1），此时线程A拿到了执行权，那么在count+1时，此时的count是为0的，因为A之前已经拿到了count是0，此时count+1=1，于是问题就产生了线程A，B都输出了1，这是因为A拿到的是已经被B修改过过期的值。

所以最简单的解决办法就是关键字**synchronized**.

```java
public class AtomicTest {

    public static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                synchronized(AtomicTest.class){
                	count++;
                    System.out.println(Thread.currentThread().getName()+"---->"+count);   
                }
            }
        });

        Thread t2 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                synchronized(AtomicTest.class){
                	count++;
                    System.out.println(Thread.currentThread().getName()+"---->"+count);   
                }
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

这样是一定可以保证原子性的，因为**synchronized**保证在操作时，其他线程不允许进入，所以代码块里的操作不会被打断。

除此之外，Java还为我们提供了强大的Atomic包，我们使用该包下的AtomicInteger来保证原子性

```java
public class AtomicIntegerDetails2 {

    private static AtomicInteger atomicInteger = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                atomicInteger.getAndAdd(1);
                System.out.println(Thread.currentThread().getName()+"----				  >"+atomicInteger.get());
            }
        });

        Thread t2 = new Thread(()->{
            for (int j = 0; j < 100; j++) {
                atomicInteger.getAndAdd(1);
                System.out.println(Thread.currentThread().getName()+"---->"+atomicInteger.get());
            }
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

## 第6章 Java多线程设计模式和算法

### 	6.1 不可变设计模式（Immutable）

1.不可变对象一定是线程安全的

2.可变对象不一定是不安全的

给一个类前加final，导致该类的对象无法被修改，就只有读操作，所以线程安全，这个方法要比加锁的性能好。不可变设计模式注意以下四点：

* 去除setter方法及所有修改自身属性的方法
* 将所有属性设置为私有
* 并用final标记
* 确保没有子类可以重载修改它的行为
* 有一个 可以创建完整对象的构造函数

```java
public final class Product{//final关键字修饰确保无子类
  private final String num;  //私有属性，不会被其他对象获取
  private final String name;
  
  public Product(String num, String name){//在创建对象时，必须指定数据，创建之后无法修改
    this.num = num;
    this.name = name;
  }
  public String getNum(){
    return num;
  }
  public String getName(){
    return name;
  }
}
```

在JDK中，像String等基本包装类型都是final的，所有实例方法均不需要同步，保证了它们在多线程情况下的性能。

### 	6.2 Future设计模式

Future模式的主要思想是异步调用，当我们需要调用一个函数时，这个函数很慢，我们又不着急这个执行结果，因此 我们可以在后台慢慢处理这个请求，对于调用者可以处理一些其他的任务，在真正需要数据的场合再去尝试获取。

对于Future模式来说，虽然他无法立即给出你需要的数据，但是它会返回一个契约给你，将来你可以凭借这个契约去重新获取你想要的数据。

​														             **Future模式的主要参与者**

| 参与者     | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| Main       | 系统启动，调用Client发出请求                                 |
| Client     | 返回Data对象，立即返回FutureData，并开启ClientThread线程装配RealData |
| Data       | 返回数据的接口                                               |
| FutureData | Future数据构造很快，但是是一个虚拟的数据，需要装配RealData   |
| RealData   | 真实数据，构造较慢                                           |

Future模式的简单实现：在实现中，有一个接口Data，这就是客户端希望获取的数据。在Future模式中，这个Data接口有两个重要的实现，一个是RealData，也就是真实数据，这就是我们最终需要获得的有价值的信息。另一个就是FutureData，它用来提取RealData的一个"订单"。因此FutureData可以立即返回。

**JDK自带Future模式：**

```java
/*
 * RealData类必须实现Callable<String>接口，	
 */
public class RealData implements Callable<String> {

    private String data;

    public RealData(String data) {
        this.data = data;
    }

    @Override
    public String call() throws Exception {
        String s = data.substring(2);
        Thread.sleep(2000);//这里模拟需要干很久的事情
        return s;
    }
}

public class FutureMain {
    public static void main(String[] args) throws InterruptedException, ExecutionException 		 {
        String s = "123";
        ExecutorService pool = Executors.newCachedThreadPool();
        FutureTask<String> futureTask = 
          new FutureTask<>(new RealData("123456"));//提交异步执行的任务
      
        pool.submit(futureTask);
        System.out.println("任务开启完成");
        Thread.sleep(2000);//模拟主程序的操作
        String result = futureTask.get();//这里去尝试获取call函数返回的数据
        result = result + s;
        System.out.println(result);
        pool.shutdown();
    }
}
```

**guava对Future模式的支持：**

```java

```

### 6.3 单例设计模式

```java
public class Singleton{//非安全，非懒加载 
  
  private Singleton(){}
  
  private static Singleton instance = new Singleton();
  
  public static Singleton getInstance(){
    return instance;
  }
}
```

```java
public class LazySingleton{//安全懒加载
  private LazySingleton{}
  
  private static synchronized LazySingleton getInstance(){
    if(instance==null){
      instance = new LazySingleton();
    }
    return instance;
  }
}
```

```java
public class StaticSingleton{//高性能安全懒加载
  private static Singleton{}
  
  private static class SingletonHodler{
    private static StaticSingleton instance = new StaticSingleton();
  }
  
  public static StaticSingleton getInstance(){
    return SingletonHodler.instance;
  }
}
```

### 6.4 生产者-消费者模式

### 6.5 并行搜索



## 第7章 多线程进阶

### 		7.1	CAS算法

​	了解这个算法之前，我们先说乐观锁和悲观锁

​	**synchronized**就是一种典型的悲观锁：这种锁的特点是，只要一拿到锁，其他的线程必须要处于阻塞状态，因为它悲观的认为并发的情况十分糟糕。

​	**CAS**锁是一种乐观锁：每次不加锁而是假设没有冲突而去完成某项操作，它乐观的认为并发状况没有那么糟糕，如果因为冲突失败就重试，直到成功为止。

​	我们之前在讨论原子性的时候，用到了AtomicInteger这个类的getAndAdd()方法，我们进去源码看看。

```java
public final int getAndAdd(int delta) {
   return U.getAndAddInt(this, VALUE, delta);
}
```

再进入**getAndAddInt(this, VALUE, delta)**这个方法的源码，我们来到了一个叫Unasfe的类，

于是我们看到了这个方法，这就是CAS算法**compareAndSetInt**，JDK8是**compareAndSwapInt**

```java
public final native boolean compareAndSetInt(Object o, long offset,
                                                 int expected,
                                                 int x);
```

但是CAS轻量级锁，会带来ABA问题。

什么是ABA问题，比如：当我们操作一个引用时A线程1进来了，线程A进行了一系列操作将A引用变为B引用，然后在变回A引用

### 7.2 Unsafe

无论遇到什么困难也不要怕微笑着面对它

## 第8章 线程池框架

## 第9章 Java并发包工具

### 9.1 ReentrantLock

**synchronized关键字的扩展：重入锁（ReentrantLock）:**

```java
public class ReentrantLock implements Runnable{
    public static ReentrantLock lock = new ReentrantLock();
    public static int i = 0;
    public void run(){
        for(int j=0;j<10000;i++){
            lock.lock();
            try{
                i++;
            }finally{
                lock.unlock();
            }
        }
    }
}
```

与关键字synchronized相比，重入锁有着明显的灵活性，何时加锁，何时放锁，重入锁之所以叫重入锁，是因为这种锁可以反复进入。当然，这种反复仅仅局限于一个线程。比如可以这样写：

```java
lock.lock();
lock.lock();
try{
    i++;
}finally{
    lock.unlock();
    lock.unlock();
}
```

一个线程连续两次获得同一把锁是允许的。

重入锁的中断响应：

对于关键字synchronized来说，如果一个线程在等待锁，那么结果只有两种情况，要么它获得这把锁继续执行，要么它就保持等待。而使用重入锁，这个线程还可以被中断。也就是在等待锁的过程中，程序可以根据需要取消对锁的请求。这个对处理死锁是有一定帮助的。

> 公平锁与非公平锁：
>
> 我们点进ReentrantLock的lock()方法，一直点到acquire()方法
>
> ```java
> public final void acquire(int arg) {
>     if (!tryAcquire(arg) &&
>         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//队列第一个节点的node对象存的Thread对象是null
>      selfInterrupt();
> }
> ```
>
> 进去tryAcquire方法，我们查看它的子类的该方法（win下ctrl+alt+B）
>
> 我们看到了熟悉的名词，公平锁（FairSync）与非公平锁（NonfairSync）
>
> 我们先来看公平锁
>
> ```java
> /**
> * Sync object for fair locks
> */
> static final class FairSync extends Sync {
>   private static final long serialVersionUID = -3000897897090466540L;
>   /**
>          * Fair version of tryAcquire.  Don't grant access unless
>          * recursive call or no waiters or is first.
>    */
>   @ReservedStackAccess
>   protected final boolean tryAcquire(int acquires) {
>       final Thread current = Thread.currentThread();
>       int c = getState();//获取锁的状态
>       if (c == 0) {//如果锁没有被持有
>           if (!hasQueuedPredecessors() &&
>               compareAndSetState(0, acquires)) {
>               setExclusiveOwnerThread(current);
>               return true;
>           }
>       }
>       //体现重入了
>       else if (current == getExclusiveOwnerThread()) {
>           int nextc = c + acquires;
>           if (nextc < 0)
>               throw new Error("Maximum lock count exceeded");
>           setState(nextc);
>           return true;
>       }
>       return false;
> }
> }
> ```
>
> ```java
> public final boolean hasQueuedPredecessors() {
>  // The correctness of this depends on head being initialized
>   // before tail and on head.next being accurate if the current
>   // thread is first in queue.
>      Node t = tail; // Read fields in reverse initialization order
>      Node h = head;
>      Node s;
>      return h != t &&
>          ((s = h.next) == null || s.thread != Thread.currentThread());
>  }
> ```
>
> 哪里公平呢？
>
> ReentranLock在加锁时执行到上面的tryAcquire方法，比如现在有一个线程被要lock()，会先定义一个变量获取一个状态值，这个状态就是现在这个锁是自由的还是被持有的，显然第一个线程进来锁一定是自由的等于0，判断c==0一定是成立的，于是向下执行，请看这个函数**!hasQueuedPredecessors()**。当第一个线程进来的时候，会先判断这个函数，这个函数的意义是判断当前队列中，有没有早早已经来等锁的线程，如果没有，那么返回false，再取非，于是**!hasQueuedPredecessors()**返回true，所有才有机会执行CAS改变状态获取锁，这时候假设我们有第二个线程来**竞争**，线程一还没有释放锁，所以状态码是1，第二个线程判断c!=0，会直接返回false，然后把它加入同步队列进行等待。
>
> 但是我们细心发现有一个else if()，这个是干嘛的呢，这就是体现了重入锁的重入特征，这个是判断当前的线程是不是重入了，重入的话就会把状态值+1。

> 非公平锁比较简单，但是就是不公平，什么是不公平呢，我们来看他的源代码
>
> ```java
> static final class NonfairSync extends Sync {
>   private static final long serialVersionUID = 7316153563782823691L;
>       protected final boolean tryAcquire(int acquires) {
>           return nonfairTryAcquire(acquires);
>       }
> }
>     //点进nonfairTryAcquire方法
>     @ReservedStackAccess
>     final boolean nonfairTryAcquire(int acquires) {
>     final Thread current = Thread.currentThread();
>     int c = getState();
>     if (c == 0) {
>       if (compareAndSetState(0, acquires)) {//这里有非公平的情况出现
>           setExclusiveOwnerThread(current);
>           return true;
>       }
>     }
>     else if (current == getExclusiveOwnerThread()) {
>       int nextc = c + acquires;
>       if (nextc < 0) // overflow
>           throw new Error("Maximum lock count exceeded");
>       setState(nextc);
>       return true;
>     }
>     return false;
> }
> ```
>
> 哪里体现了非公平呢？
>
> 我们看到了比公平锁少了**!hasQueuedPredecessors()**这个函数，那么就意味着，一个线程拿锁时，不会知道有没有比他最先来的但是已经在等待的线程，也就是说当几个线程竞争时，假定有两个线程已经被加入到了同步队列，这时候一个线程进来抢走了锁，那么前两个在队列中的线程，没错！白！等！了！，但是那个线程一来就把锁抢走了，这就体现了不公平。



下面我们来说**AbstractQueuedSynchronizer**（队列同步器）,上面的公平锁与非公平锁我们提到，当一个线程

当线程进行竞争时，会加入把没抢到锁的线程加入同步队列。我们下面来看看，大神Doug Lea是怎么将它安全的加进去的。

还是看这个方法

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//队列第一个节点的node对象存的Thread对象是null
      selfInterrupt();
}
```

这里满足**tryAcquire(arg)**等于false的时候，**!tryAcquire(arg)**会等于true，会先触发**addWaiter**方法，然后触发**acquireQueued**方法，我们会需要一个Node类。

```java
class Node{
    volatile Node prev;
    volatile Node next;
		volatile Thread thread；   
}
```

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
```

 这个方法是干什么的呢，就是将这个线程加入队列，但是加入队列的线程做尾部，它的前面，规定会有一个为Thread=null的Node。



上面有一个方法**hasQueuedPredecessors()**，上面很粗略的讲了，但是这个是要重点说的。

```java
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
 }

```

这个方法我们前面说的很简单，判断一个线程需不需要入队。这里我们来详细说，也是很难的一点。

我们前面说，当第一个线程**t1**进来的时候，执行到这个方法：h一定等于t==null，所以这个方法return false，会给这个线程加锁，返回的时候不会将这个线程入队，这是最简单的情况，说到这里，你一定想到，如果两个线程没有竞争，交替执行完成，那么这个队列绝对没用了，我们来想另外一种情况，假设有四个线程t1，t2，t3，t4：①t1没执行完没释放锁，t2来了，这是有竞争的情况，t2来判断锁的状态发现锁被持有，于是被加入队列，然后t3来了，同样，它要判断锁的状态，它很幸运，t1已经释放了锁，于是c==0，它可以去尝试拿锁，执行到这个方法时，h!=t成立，((s = h.next) == null这个是false，已经有t2在里面了，**s.thread != Thread.currentThread()**这个是重点，这段代码的意思是，判断队列中的第二个元素是不是当前线程，显然，s.thread是t2， Thread.currentThread()是t3，所以整个方法返回true，所以返回回去我们可以看到，t3不执行加锁并且入队，说到这里，大概总结一下，就是说队列中已经有了正在等待的t2，那么人家一定是先来的，要加锁也轮不到你t3，所以你就乖乖加入t2的后面去阻塞。②t2在判断锁的状态前，t1把锁释放了，当t3

### 9.2 Condition

之前的wait和notify只能配合synchronized关键字使用。

Condition是与重入锁相关联的。通过Lock接口（重入锁实现了这个接口）的newCondition()方法生成一个与当前重入绑定的Condition实例。利用Condition对象，我们就可以让线程在合适的时间做合适的事情。

await()方法会使当前线程等待，同时释放当前锁，知道其他线程使用singal或singalAll，线程会重新获得锁并继续执行，当线程被中断时，也能跳出等待。

awaitUninterruptibly方法不会在等待过程中响应中断

singal方法会唤醒一个正在等待的线程，singalAll是唤醒全部

### 9.3 ReentrantReadWriteLock

读写分离锁，这种锁会有效的减少锁的竞争，提升系统的性能。

运行下面的代码：你会发现程序运行的时间大概在20秒，因为我在程序中使用的是普通的重入锁，所以，读读，读写，写读，写写之间都会被阻塞，每一个都要睡眠一秒才能让下一个来执行。

 将注释部分取消，我们会发现程序的运行时间在2秒左右，所以读多写少的环境下，读写锁会很大提升系统的性能。

我们借用CountDownLatch来让主线程停着方便记录时间：

```java
import java.util.Random;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLock {

    private static Lock lock = new ReentrantLock();

    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    private static Lock readLock = readWriteLock.readLock();

    private static Lock writeLock = readWriteLock.writeLock();

    private int value;

    public int isReading(Lock lock) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);
            return value;
        }finally {
            lock.unlock();
        }
    }

    public int isWriting(Lock lock, int value) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);
            this.value = value;
            return value;
        }finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ReadWriteLock readWriteLock = new ReadWriteLock();
        Runnable read = () -> {
            try {
                //readWriteLock.isReading(readLock);
                readWriteLock.isReading(lock);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        Runnable write = () -> {
            try {
                //readWriteLock.isWriting(writeLock, new Random().nextInt());
                readWriteLock.isWriting(lock, new Random().nextInt());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        for (int i = 0; i < 18; i++) {
            new Thread(read).start();
        }

        for(int i = 18;i<20;i++){
            new Thread(write).start();
        }
    }

}

```

运行结果：

```java
//使用读写锁
3023ms
//不使用读写锁
20018ms
```

**锁降级**：

锁降级是指写锁降级为读锁。锁降级是指把当前拥有的写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。

写锁要在读锁之前获取。



**读写锁的实现分析：**

在ReentranLock

### 9.4 CountDownLatch

倒数计时器，它的作用类似与join，但是它的功能比join丰富的多，它提供了一种带参的构造方法，参数是int count，它是怎么实现类似join的功能的呢，它有一个方法countDown()，这个方法的作用是每次调用它，都会使count-1，它还有一个方法await()，这个方法是阻塞当前线程，直到count=0；才会执行await()之后的代码。

我们来一个小栗子练习一下：

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * CountDown
 */
public class CountDownLatchExa implements Runnable{

    private static CountDownLatch countDownLatch = new CountDownLatch(10);

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName()+"完成任务");
        countDownLatch.countDown();
    }

    public static void main(String[] args) throws InterruptedException {
        CountDownLatchExa countDown = new CountDownLatchExa();
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.submit(countDown);
        }
        countDownLatch.await();
        System.out.println("所有任务结束");
    }
}

```

运行结果：我们发现await()的作用就是阻塞，直到count=0

```java
pool-1-thread-2完成任务
pool-1-thread-8完成任务
pool-1-thread-9完成任务
pool-1-thread-5完成任务
pool-1-thread-7完成任务
pool-1-thread-4完成任务
pool-1-thread-10完成任务
pool-1-thread-6完成任务
pool-1-thread-1完成任务
pool-1-thread-3完成任务
所有任务结束
```

### 9.5 CyclicBarrier

它有两个构造函数

```java
public CyclicBarrier(int parties, Runnable barrierAction)
public CyclicBarrier(int parties)
```

有一个参数的表示：设置屏障拦截的线程数量，在最后一个await()的线程达到屏障时，屏障开通；

例如下面的代码，一定会打印三句话，但是最后一句一定是“屏障已经开通”，前面两句话的顺序不定，因为我设置的最大的阻塞线程数是2，有可能Thread0先被阻塞，main线程后被阻塞，也有可能相反，但是我们可以看出当阻塞的线程数达到2的时候，屏障就开通了，程序就会接着往下进行。

```java
package com.Concurrency.currentUtil;

import java.util.concurrent.*;

public class CyclieBarrierExample{

    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

    public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
        new Thread(){
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName()+"已经被屏障阻塞");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }.start();
        System.out.println(Thread.currentThread().getName()+"已经被屏障阻塞");
        cyclicBarrier.await();
        System.out.println("屏障已经开通");
    }
}

```

有两个参数的表示：在最后一个await()的线程达到屏障时，屏障开通，会先执行barrierAction中的方法，然后在执行开通后的线程。

还是阻塞两个线程，我们发现在屏障打开前，两个线程还是阻塞状态，但是当屏障一旦打开，会先执行，参数中的对象的run()方法，哪个线程执行这个方法呢，最后被阻塞的线程醒来先执行这个对象的方法，然后向下执行程序。

```java
package com.Concurrency.currentUtil;

import java.util.concurrent.*;

public class CyclieBarrierExample{

    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new Demo());

    public static void main(String[] args){
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName()+"已经被屏障阻塞");
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println("屏障已经放开"+Thread.currentThread().getName()+"已经执行");
            }
        }).start();

        try {
            System.out.println(Thread.currentThread().getName()+"已经被屏障阻塞");
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println("屏障已经放开"+Thread.currentThread().getName()+"已经执行");
    }

    static class Demo implements Runnable{
        @Override
        public void run() {
            System.out.println("屏障已经放开由"+Thread.currentThread().getName()+"先执行");
        }
    }
}

```

我们来一个例子练习一下

```java
import java.util.Map;
import java.util.concurrent.*;

/**
 * 用CyclieBarrier组件模拟银行流水计数
 * 需求：假设有5个线程处理五个库中的银行钱数，最后一个线程进行汇总给出总数
 */
public class CyclieBarrierExample implements Runnable{
    private ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

    private Executor ecu = Executors.newFixedThreadPool(5);

    private CyclicBarrier cy = new CyclicBarrier(5, this);

    public void count(){
        for (int i = 0; i < 5; i++) {
            ecu.execute(new Runnable() {
                @Override
                public void run() {
                    map.put(Thread.currentThread().getName(), 1);
                    try {
                        cy.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void run() {
        int result = 0;
        for (Map.Entry<String, Integer> m:map.entrySet()){
            result += m.getValue();
        }
        map.put("result", result);
        System.out.println(result);
    }

    public static void main(String[] args) {
        CyclieBarrierExample cyclieBarrierExample = new CyclieBarrierExample();
        cyclieBarrierExample.count();
    }
}

```

### 9.6 Semaphore

信号量可以控制访问特定资源的线程数量，保证合理的使用公共资源。

我们打个比方，Semaphore是交通信号灯，当它规定只有10辆车能走时，就说明只有十个车能同时在一个马路上行驶，当有几个车驶离这个马路时，那么其他的车就可以进入这个马路，就是有再多的车，我们通过交通信号灯就可以保证整条马路总是只有10辆车行驶，直到没有车。

下面的代码中，我们声明的线程池中有30个线程，但是每次只会允许10个并发执行。

```java
package com.Concurrency.currentUtil;

import java.util.concurrent.*;

public class CyclieBarrierExample{

    private static final int THREAD_COUNT = 30;

    private static ExecutorService executorService = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore semaphore = new Semaphore(10);

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < THREAD_COUNT; i++) {
                    executorService.execute(new Runnable() {
                        @Override
                        public void run() {
                            try {
                                semaphore.acquire();
                            } catch (InterruptedException e) {

                            }
                            System.out.println("doing something");
                            semaphore.release();
                        }
                    });
                }
            }
        });
        executorService.shutdown();
    }
}

```



### 9.7 Guava----RateLimiter

​	日常生活中的应用都有一定的访问速率和上限，如果请求的速率超过上限，很可能会压垮系统，多余的请求也无法处理，RateLimiter就是这样一款限流的工具。

​	限流是有算法的，RateLimiter使用的是令牌桶算法。

​	在这个算法中，假设有一个桶，里面存放令牌，处理程序只有拿到令牌后，才可以对请求进行处理，如果没有可用的令牌，要么等，要么丢弃请求。为了限制流速，这个算法可以在固定时间内在桶中产生几个令牌（这个根据需求定），比如你限定应用每秒处理一个请求，那么就让桶每秒产生一个令牌，桶是有容量限制的，桶满丢弃令牌，空桶不处理请求。我们看下面的代码：

```java
import com.google.common.util.concurrent.RateLimiter;

public class RateLimiterDemo {

    private static RateLimiter rateLimiter = RateLimiter.create(2);//参数的含义是每秒处理几个																	请求

    public static class Task implements Runnable{

        @Override
        public void run() {
            System.out.println(System.currentTimeMillis());
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 50; i++) {
            rateLimiter.acquire();//通过acquire方法获取令牌
            new Thread(new Task()).start();
        }
    }
}
```

运行部分结果：我们可以看出来，每秒至多处理两个请求，令牌没拿光，但是1s还没结束，其他的线程只能去等待

```java
1570521115242
1570521115742
1570521116241
1570521116742
```

但是在很多时候，我们不需要等待，直接丢掉过载的请求，这样用户的体验会更好，所以我们使用tryAcquire()。

我们对上面的for循环进行修改

```java
for (int i = 0; i < 50; i++) {
    if(!rateLimiter.tryAacquire()){//如果没法获取令牌
        continue;
    }
    new Thread(new Task()).start();
 }
```

运行结果：因为for循环50次很快，我们虽然规定1s产生两个令牌，500ms产生一个令牌，所以50次很可能没有1s就执行完了，所以第一个拿到令牌的输出，其他的都请求都舍弃了。你可以将其改成死循环，这样结果一定不一样，但是同样的是一秒只会至多两个请求

```
1570524145713
```

## 第10章  Java中的线程池

线程的创建和销毁是耗费资源的操作，Java线程依赖于内核线程，创建线程要进行操作系统状态的切换

### 10.1 核心线程池的内部实现

```java
public ThreadPoolExecutor(int corePoolSize,//指定线程池中的线程数量
                              int maximumPoolSize,//指定线程池中的最大线程数量
                            long keepAliveTime,//超过corePoolSize的空闲线程，多长时间内会被销毁
                              TimeUnit unit,//keepAliveTime单位
                              BlockingQueue<Runnable> workQueue//任务队列被提交但未执行的任务
                         ThreadFactory threadFactory,//用于创建线程的工厂
                              RejectedExecutionHandler handler//任务太多可以拒绝，拒绝策略
                              ) {
}
```

详细说明**workQueue**和**handler**：

**workQueue**指

 ThreadPoolExecutor执行executor()方法时，分为四种情况：

1. 如果当前运行线程少于corePoolSize，则创建新线程来执行任务（需要获取全局锁）
2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue.
3. 如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（执行这一步需要获取全局锁）
4. 如果创建线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectExecution()方法

采用上述方法的总体设计思路，是为了在执行execute()方法时，尽可能地避免获取全局锁，线程池完成预热之后（当前运行线程数大于等于corePoolSize），几乎所有的execute()方法调用都是执行步骤2。

### 10.2 各种样式的线程池

Executor框架为我们提供了各种类型的线程池，主要有以下几个方法

```java
public static ExecutorService newFixedThreadPool(int nThreads);
public static ExecutorService newCachedThreadPool();
public static ExecutorService newSingleThreadExecutor(int nThreads);
public static ScheduledExecutorService newSingleThreadScheduledExecutor(int nThreads);
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize);
```

**1.newFixedThreadPool()方法**：创建一个有固定数量线程的线程池，有多余的任务执行，请等待，有空闲线程时才可以执行。

我们创建5个线程但是提交10次任务，我们发现只有5个线程在执行任务，执行完成之后才会执行下一次任务的执行会在 1秒后开始，还是上一次的5个线程执行了这个任务。

```java

/*
 * 我们在线程池中创建5个线程，但是提交10次任务
 */
public class ThreadPoolDemo {

    static class Task implements Runnable{
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Task t = new Task();
        ExecutorService es  = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 10; i++) {
            es.submit(t);
        }
        Thread.sleep(2000);
    }
}
//执行结果
1576493503217:Thread ID:12
1576493503217:Thread ID:14
1576493503217:Thread ID:13
1576493503217:Thread ID:11
1576493503217:Thread ID:15
----------重新执行----------
1576493504221:Thread ID:12
1576493504222:Thread ID:15
1576493504222:Thread ID:14
1576493504225:Thread ID:13
1576493504225:Thread ID:11
```

**2.newCachedThreadPool()方法**：创建一个无固定数量线程的线程池，有空余线程则执行任务，没有则新创建线程。

上面的代码我们不动，把newFixedThreadPool(5)换为newCachedThreadPool()，我们发现任务都是一起完成，提交10次任务，它会创建10个线程依次执行任务。

```java
1576492068691:Thread ID:11
1576492068691:Thread ID:13
1576492068691:Thread ID:14
1576492068691:Thread ID:12
1576492068691:Thread ID:15
1576492068691:Thread ID:16
1576492068691:Thread ID:17
1576492068691:Thread ID:18
1576492068691:Thread ID:19
1576492068691:Thread ID:20
```

**3.newSingleThreadExecutor()方法**：创建一个一个线程的线程池。有多余任务时加入任务队列，按先来后到执行

```java
public class ThreadPoolDemo {

    static class Task implements Runnable{
        private static int i = 0;
        @Override
        public void run() {
            System.out.println("第"+i+"个被加入任务队列"+System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            i = i+1;
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Task t = new Task();
        ExecutorService es  = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 10; i++) {
            es.submit(t);
        }
        Thread.sleep(10000);
        es.shutdown();
    }
}
//输出结果
第0个被加入任务队列1576548776820:Thread ID:11
第1个被加入任务队列1576548777821:Thread ID:11
第2个被加入任务队列1576548778825:Thread ID:11
第3个被加入任务队列1576548779827:Thread ID:11
第4个被加入任务队列1576548780832:Thread ID:11
第5个被加入任务队列1576548781833:Thread ID:11
第6个被加入任务队列1576548782835:Thread ID:11
第7个被加入任务队列1576548783838:Thread ID:11
第8个被加入任务队列1576548784840:Thread ID:11
第9个被加入任务队列1576548785842:Thread ID:11
```

线程池只创建一个线程，一个线程被提交10次任务，  

**4.newSingleThreadScheduledExecutor()方法**：扩展了给定时间执行某任务的功能，某个固定的延时之后执行，或者周期性执行某个任务。

```java

```

**5.newScheduledThreadPool()方法**：可以指定线程数量。

```java
public class ThreadPoolScheduled {
    public static void main(String[] args) {
        ScheduledExecutorService ses = Executors.newScheduledThreadPool(10);
        ses.scheduleAtFixedRate(
                new Runnable() {
                    @Override
                    public void run() {
                        try {
                            Thread.sleep(1000);
                            System.out.println(System.currentTimeMillis()/1000);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }, 0, 2, TimeUnit.SECONDS
        );
    }
}
//执行的部分结果
1576498195
1576498197
1576498199
1576498201
1576498203
1576498205
```

我们将周期设置为2，每个任务执行的时候都会先睡眠一秒，所以一个任务执行结束的时候，2秒的周期还没有到，所以它会等2秒的周期到了后，再执行下一次任务，所以依次任务执行完到下一次任务执行之间都是间隔2秒。如果我们将scheduleAtFixedRate方法换为scheduleWithFixedDelay方法，结果就是间隔都是3秒，因为周期 不变是2秒，但是任务执行1秒，在任务执行完毕后，过2秒的周期，最后执行下一次任务，所以它的时间间隔3秒。 

### 10.3 自定义线程池创建

线程池的主要作用是为了线程复用，也就是避免了线程的频繁创建。但是，最开始的那些线程从何而来？答案就是ThreadFactory。

ThreadFactory是一个接口，它只有一个用来创建线程的方法。

```java
Thread newThread(Runnable r);
```

当线程池需要新建线程时，就会调用这个方法。

我们来手动设置一下我们要自定义的线程池，这里的构造函数我们没必要设置拒绝策略，因为设置的任务队列是SynchronousQueue<>，这是一个无容量的任务队列

```java
import java.util.concurrent.*;

public class ThreadPoolDemo {

    static class Task implements Runnable{
        @Override
        public void run() {
            System.out.println(System.currentTimeMillis()+":Thread ID:"+Thread.currentThread().getId());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Task t = new Task();
        ExecutorService es = new ThreadPoolExecutor(
                5,	//指定线程池中的线程数量
                5,	//指定线程池最大的线程数量
                0L, //超过corePoolSize的空闲线程，超过多久会被销毁
                TimeUnit.MILLISECONDS,//keepAliveTime的单位
                new SynchronousQueue<>(),//任务队列，被提交但未被执行的任务
                new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r);
                        t.setDaemon(true);
                        System.out.println("creat"+t.getName());
                        return t;
                    }
                }//线程工厂，用于创建线程，一般用默认。
          
          			//拒绝策略，任务太多来不及处理时，如何拒绝任务
        );
        for (int i = 0; i < 10; i++) {
            es.submit(t);
        }
        Thread.sleep(2000);
    }
}

```

运行结果：

```java
creatThread-0
creatThread-1
creatThread-2
creatThread-3
creatThread-4
1570692822181:Thread ID:13
1570692822190:Thread ID:15
1570692822184:Thread ID:17
1570692822181:Thread ID:14
1570692822183:Thread ID:16
```

#### 10.3.1 自定义线程池的重要参数

runnableTaskQueue(任务队列)：用于存放任务的阻塞队列

* ArrayBlockingQueue：这是一个基于数组的有界阻塞队列，FIFO原则对元素进行排序
* LinkedBlockingQueue：基于链表的阻塞队列，吞吐量高于上面的ArrayBlockingQueue
* SynchronousQueue：不存储元素的阻塞队列，每个插入操作必须等待另一个线程调用移除操作，否则插入操作一直处于阻塞状态。
* PriorityBlockingQueue：一个具有优先级的无限阻塞队列

RejectedExecutionHandler(饱和策略)：当队列和线程池都饱和了，需要一种策略处理提交的新任务。默认情况下是AbortPolicy，表示无法处理新任务时抛出的异常。

* AbortPolicy：直接抛出异常
* CallerRunsPolicy：只用调用者的线程来运行任务
* DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务
* DiscardPolicy：不处理，丢弃掉

### 10.4 扩展线程池

 

### 10.5 优化线程池线程数量



### 10.6 在线程池中寻找堆栈



### 10.7 分而治之：Fork/Join框架

fork()表示会创建一个线程，join()就是等待。

在JDK中，给出一个ForkJoinPool线程池，对于fork()方法并不着急开启线程，而是提交给这个线程池进行处理。

ForkJoinPool线程池一个重要的接口：

```java
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task)
```

你可以向ForkJoinPool线程池提交一个ForkJoinTask任务。所谓ForkJoinTask任务就是支持fork()方法分解及join()方法等待任务。ForkJoinTask任务有两个重要的子类，RecursiveAction（无返回值）类和RecursiveTask（返回V类型）类。

下面的样例求阶乘：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;

/**
 * 求阶乘，我们模拟将任务分成6份去执行
 */
public class ForkJoinEmaple extends RecursiveTask<Integer> {

    private static final Integer THRES_NUM = 2;

    private Integer start;

    private Integer end;

    public ForkJoinEmaple(Integer start, Integer end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        Integer sum = 1;
        if(end-start<THRES_NUM){
            for(int i=start;i<=end;i++){
                sum*=i;
            }
        }else{
            Integer step = end/6;
            Integer pos = start;
            List<ForkJoinEmaple> list = new ArrayList<>();
            for (int i = 0; i < 6; i++) {
                Integer lastOne = pos+1;
                ForkJoinEmaple f = new ForkJoinEmaple(pos, lastOne);
                pos += step;
                list.add(f);
                f.fork();
            }
            for(ForkJoinEmaple f:list){
                sum*=f.join();
            }
        }
        return sum;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ForkJoinPool pool = new ForkJoinPool();
        ForkJoinEmaple forkJoinEmaple = new ForkJoinEmaple(1, 12);
        ForkJoinTask<Integer> forkJoinTask = pool.submit(forkJoinEmaple);
        System.out.println(forkJoinTask.get());
    }
}
```

当我们不停分割当任务足够小时，就会执行if块内的代码，每个子任务调用fork()方法时，又会进入compute方法，看看当前的子任务还需不需要继续分解。

**异常处理**：

ForkJoinTask在执行时可能会抛出异常，但是我们没办法在主线程中直接捕获，所以它提供了isCompletedAbnormally()方法来检查任务是否已经抛出异常或被取消了，并通过ForkJoinTask的getException方法捕获异常。

```java
if(task.isCompletedAbnormally()){
    sout(task.getException());
}
```

**实现原理**：



### 10.8 Google-Guava中对线程池的扩展



## 第11章  JDK的并发容器

### 11.1 ConcurrentHashMap

JDK1.7 HashMap：1.7的put方法第一步是先给Entry数组一个初始化的，如果你的初始化

1.7的不安全发生在扩容

```java
void addEntry(int hash, K key, V value, int bucketIndex){
  if((size>=threshold)&&(null != table[bucketIndex])){
    resize(2*table.length);
    hash = (null!=key)?hash(key):0;
    bucketIndex = indexFor(hash, table.length);
	}
}
```

扩容前

```java
for(Entry<K,V> e:table){
  while (e != null){
        Entry<K,V> next = e.next; // <--假设线程一执行到这里就被调度挂起了
        e.next = newTable[i];
        newTable[i] = e;
        e = next;
    } 
}		
```

ConcurrentHashMap：高效的并发HashMap

聊ConcurrentHashMap之前，我们先回顾HashMap，这里有几个问题：

* 为什么默认的加载因子是0.75？

0.5：空间利用率低

1：查询效率低，Hash碰撞多

* Java8以后，为什么链表长度超过8转红黑树？

* 初始容量为什么必须是2的指数次幂？

在HashMap中，我们的map的初始容量必须是2的指数次幂，hash函数运算的方法是HashCode计算出来的值H，

H & （16-1）16为初始化数组长度，初始容量为2的指数次幂可以防止大量的Hash碰撞。

### 11.2 CopyOnWriteArrayList

CopyOnWriteArrayList：在读多写少的场合，这个List性能非常好

### 11.3 ConcurrentLinkedQueue

ConcurrentLinkedQueue：高效的并发队列，可以看作是一个线程安全的LinkedList。

## 第12章 JDK1.6以后对Synchronized的优化及注意事项

### 12.1 提高锁性能

#### 12.1.1 减少锁持有时间

有下面的代码：

假设method1和method2是重量级的方法，但是只有mutexMethod才用同步控制，那么这个方法的性能就很低

```java
public synchronized void synMethod(){
	method1();
    mutexMethod();
    method2();
}
```

我们进行如下的修改：锁的占用时间减少了，性能会大大增加。

```java
public void synMethod(){
	method1();
    synchronized(this){
        mutexMethod();
    }
    method2();
}
```

还有一个例子就是double check，在这个线程安全的单例模式中，如果进行一次判断不管对象有没有被初始化，都需要进行先加锁后判断，所以性能会降低，double check后，先判断对象有没有被初始化，没有的话再加锁。这样性能就会更好。

#### 12.1.2 减小锁粒度

这种方案典型的案例是ConcurrentHashMap，ConcurrentHashMap中包含了许多segment，默认分为16个段，

当需要在ConcurrentHashMap中新增一个表项时，并不是对整个map加锁，而是首先根据hashcode得到该表项应该被存放到哪个段中，然后对该段加锁，并完成put()操作，在多线程环境中，如果多个线程同时进行put()方法操作，只要不存放在同一个段中，线程之间便可以做到真正的并行。

锁的粒度减小也会带来一些问题：当系统需要获取全局锁的时候，其消耗的资源会比较多，比如，在求ConcurrentHashMap的size时，它要对所有的段进行加锁，然后才可以统计总数。

当然，在计数时，ConcurrentHashMap会先尝试两次通过不锁住segment的方式计数，如果统计的过程中，count发生了变化，那么再采用加锁的方式来统计各个segment的大小。

不管怎样，ConcurrentHashMap的size要比HashMap的size性能低。

#### 12.1.3 用读写分离锁来替换独占锁

ReadWriteLock可以提高性能，用读写分离锁来替换独占锁是减小锁粒度的一种特殊情况。

在读多写少的环境中，读写锁占绝对优势，如果在读写数据时均只用独占锁，两种操作之间都要互相等待，允许多线程之间互读，单线程写，这会大大提升性能

#### 12.1.4 锁分离

上面说了读写锁，延伸一下就是锁分离。

在LinkedBlockingQueue的实现中，take()和put()分别实现了从队列中取得数据和往队列中增加数据功能。

由于LinkedBlockingQueue是基于链表的，因此两个操作分别作用于队列的进行了修改操作，分别作用于队列的前端和尾端。从理论上讲，两者并不冲突。

如果要使用独占锁，那么就要求在两个操作进行时获取当前队列的独占锁，那么take()方法和put()方法就不可能真正的并发，运行时，它们会彼此等待对方释放锁资源，在这种情况下，锁竞争会相对比较激烈。影响性能

所以在JDK中是把两种操作的锁分离开。

```java
private final ReentrantLock takeLock = new ReentrantLock();//take()方法需要持有的锁

private final ReentrantLock putLock = new ReentrantLock();//put()方法需要持有的锁
```

所以take和put操作就此相互独立，它们之间不存在锁竞争的关系，

#### 12.1.5 锁粗化

为了保证多线程间有效的并发，会要求每个线程持有锁的时间尽量短，即在使用完公共资源后，应该立即释放锁。只有这样其他等待的锁才可以获取资源任务，但是，如果对同一个锁不停进行请求，同步和释放，其本身也会消耗资源。

例如：这种在循环中加锁的形式，会十分耗费资源，因为会不断对一个锁进行请求

```java
for(int i=0;i<10;i++){
    synchronized(lock){
        
    }
}
```

我们做如下修改

```java
synchronized(lock){
    for(int i=0;i<10;i++){
        
    }
}
```

### 12.2 虚拟机对锁的优化

无锁---》偏向锁---》轻量级锁---》重量级锁

#### 12.2.1 锁偏向

对于**几乎没有锁竞争**的场合，一个线程获取了锁，锁进入偏向模式。当这个线程再次请求时，无须再做任何同步操作，这样就节省了有关锁的申请动作，但是在锁竞争很激烈的环境中，偏向模式很大可能会失效，使用虚拟机参数-XX:+UseBiasedLocking可以开启偏向锁。偏向锁主要是同一个线程进入代码块才会对性能有优化

偏向锁的原理：

当锁对象第一次被线程获取，虚拟机将会把对象头中的标志位设为“01”，即偏向模式，使用CAS将ThreadID改为当前线程，以后进入和退出代码块的时候不需要进行CAS操作。

#### 12.2.2 轻量级锁

如果偏向锁失败，那么虚拟机并不会立即挂起线程，它还会使用一种称为轻量级锁的优化手段。它将对象头部作为指针指向持有锁的线程堆栈内部来判断一个线程是否持有对象锁。如果线程获得轻量级锁成功，就可以顺利进入临界区。失败，被其他线程抢了锁，当前线程膨胀为重量级锁。轻量级锁的好处就是在多线程交替执行的环境下，避免重量级锁带来的性能消耗。

#### 12.2.3 自旋锁

锁膨胀后，为了避免线程真实地在操作系统层面挂起，虚拟机会做几个空循环（自旋），经过循环后如果可以获得到锁，就顺利进入临界区。如果不行，将在操作系统层面挂起。默认是自旋10次，可以通过参数：-XX:PreBlockSpin来更改

#### 12.2.4 锁消除

会将无用的无竞争的锁去除掉，比如：我们很有可能在没有并发的情况下使用StringBuffer或者Vector，这些类内部的加锁同步都是没有必要的。这些就是要去除掉的。

### 12.3 ThreadLocal

除了控制资源的访问外，我们还可以通过增加资源来保证所有对象的线程安全，100个人抢一个笔去填单子，我们可以增加笔的数量

如果说锁的使用是第一种思路，那么ThreadLocal使用的就是第二种思路

#### 12.3.1 ThreadLocal的使用



## 第13章 
