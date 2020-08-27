# 1.2 Java多线程

> 作者：SunSAS
>
> **介绍：** 关于多线程，推荐去看《Java并发编程的艺术》这本书，当然这本书涉及比较深，一般是需要反复读上即便才能理解，这篇也是基于此书，不过有很多东西太过底层，或由于面试工作中基本不会过问，我这里也不会列出。这里推荐一位大佬的专栏[透彻理解Java并发编程](https://segmentfault.com/blog/ressmix_multithread),非常底层，就是太多了。

## 1.2.1 Java多线程基础

### 1. 进程和线程
**程序** (program) 是为完成特定任务、用某种语言编写的一组指令的集合。即指一段静态的代码，静态的对象。

**进程**（Process）**是计算机中的程序关于某数据集合上的一次运行活动**，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。在早期面向进程设计的计算机结构中，进程是程序的基本执行实体；在当代面向线程设计的计算机结构中，进程是线程的容器。程序是指令、数据及其组织形式的描述，进程是程序的实体。

**线程**，有时被称为轻量级进程（Lightweight Process,LWP），是程序执行流的最小单元。线程是程序中一个单一的顺序控制流程。进程内一个相对独立、可调度的执行单元，**是系统独立调度和分派CPU的基本单位**，也指运行中的程序的调度单位。在单个程序中同时运行多个线程完成不同的工作，称为多线程。


### 2. 并行和并发
**并发**（concurrency）：指在同一时刻只能有一条指令执行，但多个进程指令被快速的轮换执行，使得在宏观上具有多个进程同时执行的效果，但在微观上并不是同时执行的，只是把时间分成若干端，使多个进程快速交替的执行。  
![并发](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200730/20160522231219254.jpg)

**并行**（parallellism）:指在同一时刻，有多条指令在多个处理器上同时执行   
![并行](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200730/20160522231156816.jpg)  


通过多线程实现并发，并行：
- Java中的Thread类定义了多线程，通过多线程可以实现并发或并行。
- 在CPU比较繁忙，资源不足的时候（开启了很多进程），操作系统只为一个含有多线程的进程分配仅有的CPU资源，这些线程就会为自己尽量多抢时间片，这就是通过多线程实现并发，线程之间会竞争CPU资源争取执行机会。
- 在CPU资源比较充足的时候，一个进程内的多线程，可以被分配到不同的CPU资源，这就是通过多线程实现并行。
- 至于多线程实现的是并发还是并行？上面所说，所写多线程可能被分配到一个CPU内核中执行，也可能被分配到不同CPU执行，分配过程是操作系统所为，不可人为控制。所以，如果有人问我我所写的多线程是并发还是并行的？我会说，都有可能。
- 不管并发还是并行，都提高了程序对CPU资源的利用率，最大限度地利用CPU资源


### 3. 为什么要使用多线程
> 来源于《Java并发编程的艺术》第四章

1. **更多的处理器核心**  
随着处理器的核心数量越来越多，以及超线程技术的广泛应用，现在大多数的计算机都比以往更擅长并行计算，而处理器性能的提升方式，也从更高的主频向更多的核心发展。如何利用好处理器上的多个核心也成了现在的主要问题。  
线程是大多数系统调度的基本单元，一个程序作为一个进程来运行，程序运行的过程中能够创建多个线程，而一个线程在同一时刻只能运行在一个处理器核心上。试想一下，一个单线程程序在运行时只能使用一个处理器核心，那么再多的处理器核心也无法显著提高该程序的执行效率。相反，如果该程序使用了多线程技术，将计算逻辑分配到多个处理器核心上，就会**显著减少程序处理的时间，并且随着更多处理器核心的加入而变得更有效率**

2. **更快的响应时间**  
有时我们会编写一些较为复杂的代码（这里的复杂不是说复杂的算法，而是**复杂的业务逻辑**），例如，一笔订单的创建，它包括插入订单数据、生成订单快照、发送邮件通知卖家和纪录货品销售数量等。用户从单机“订购”按钮开始，就要等待这些操作全部完成才能看到订购成功的结果。但是这么多业务操作，如何能够更快地完成呢？  
在上面的场景中，可以使用多线程技术，即将**数据一致性不强的操作派发给其他线程处理（也可以使用消息队列）**，如生成订单快照、发送邮件等。这样做的好处是响应用户请求的线程能够尽可能快的处理完成，缩短了响应时间，提升了用户体验。

3. **更好的模型编程**  
Java 为多线程提供了良好、考究并且一致的编程模型，是开发人员能够更加专注于问题的解决，即为所遇到的问题建立适合的模型，而不是绞尽脑汁地考虑如何将其多线程化。一旦开发人员建立好了模型，稍作修改总是能够方便地映射到Java提供的多线程模型上。

### 4. 创建线程
1. **继承Thread类**，不建议使用, 因为Java是单继承的，继承了Thread就没办法继承其它类了，不够灵活
2. **实现Runnable接口**，比Thread类更加灵活，没有单继承的限制
3. **实现Callable接口**，Callable是重写的call()方法并且有返回值并可以借助FutureTask类来判断线程是否已经执行完毕或者取消线程执行
4. **使用线程池创建**

#### 4.1 继承Thread类
继承Thread类创建线程的步骤为：
1. 创建一个类继承Thread类，重写run()方法，将所要完成的任务代码写进run()方法中；
2. 创建Thread类的子类的对象；
3. 调用该对象的start()方法，该start()方法表示先开启线程，然后调用run()方法；

#### 4.2 实现Runnable接口

1. 创建一个类并实现Runnable接口
2. 重写run()方法，将所要完成的任务代码写进run()方法中
3. 创建实现Runnable接口的类的对象，将该对象当做Thread类的构造方法中的参数传进去
4. 使用Thread类的构造方法创建一个对象，并调用start()方法即可运行该线程

> Thread是实现自Runable接口

```java
// 变体写法
public static void main(String[] args) {
    // 匿名内部类
    new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId());
        }
    }).start();

    // 尾部代码块, 是对匿名内部类形式的语法糖
    new Thread() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId());
        }
    }.start();

    // Runnable是函数式接口，所以可以使用Lamda表达式形式
    Runnable runnable = () -> {System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId());};
    new Thread(runnable).start();
}
```

#### 4.3 实现Callable接口
1. 创建一个类并实现Callable接口
2. 重写call()方法，将所要完成的任务的代码写进call()方法中，需要注意的是call()方法有返回值，并且可以抛出异常
3. 如果想要获取运行该线程后的返回值，需要创建Future接口的实现类的对象，即FutureTask类的对象，调用该对象的get()方法可获取call()方法的返回值
4. 使用Thread类的有参构造器创建对象，将FutureTask类的对象当做参数传进去，然后调用start()方法开启并运行该线程。

> 当线程不需要返回值时使用Runnable，需要返回值时就使用Callable，一般情况下不直接把线程体代码放到Thread类中，一般通过Thread类来启动线程

#### 4.4 使用线程池创建
1. 使用Executors类中的newFixedThreadPool(int num)方法创建一个线程数量为num的线程池
2. 调用线程池中的execute()方法执行由实现Runnable接口创建的线程；调用submit()方法执行由实现Callable接口创建的线程
3. 调用线程池中的shutdown()方法关闭线程池

### 5. 线程的状态
Java线程在运行的生命周期中可能处于下表所示的6中不同的状态，在给定的一个时刻，线程只能处于其中一个状态： 

状态名称 |	说明
--- | ---
NEW  |	初始化状态,线程被创建，但是还没有调用 start()方法
RUNNABLE | 	运行状态，Java线程将操作系统中的就绪和运行两种状态笼统地称作“运行中”
BLOCKED  |	阻塞状态，表示线程阻塞于锁
WAITING  |	等待状态，表示线程进入等待状态，进入等待状态表示当前线程需要等待其他线程做出一些特定地动作（通知或中断）
TIME_WAITING | 	超时等待状态，该状态不同于WAITING，它是可以在指定的时间自动返回的
TERMINATED  |	终止状态，表示当前线程已经执行完毕

![线程的状态](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200730/20190105211024300.png)


注意点：
1. Thread.sleep(long millis)，一定是当前线程调用此方法，**当前线程进入TIMED_WAITING状态，但不释放对象锁**，millis后线程自动苏醒进入就绪状态。作用：给其它线程执行机会的最佳方式。
2. Thread.yield()，一定是当前线程调用此方法，**当前线程放弃获取的CPU时间片，但不释放锁资源，由运行状态变为就绪状态**，让OS再次选择线程。作用：让相同优先级的线程轮流执行，但并不保证一定会轮流执行。实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。Thread.yield()不会导致阻塞。该方法与sleep()类似，只是不能由用户指定暂停多长时间。
3. t.join()/t.join(long millis)，当前线程里调用其它线程t的join方法，**当前线程进入WAITING/TIMED_WAITING状态，当前线程不会释放已经持有的对象锁**。线程t执行完毕或者millis时间到，当前线程进入就绪状态。
4. obj.wait()，当前线程调用对象的wait()方法，**当前线程释放对象锁，进入等待队列**。依靠notify()/notifyAll()唤醒或者wait(long timeout) timeout时间到自动唤醒。
5. obj.notify()唤醒在此对象监视器上等待的单个线程，选择是任意性的。notifyAll()唤醒在此对象监视器上等待的所有线程。注意被**唤醒的线程不一定是运行状态，也可能是就绪状态**，等待cpu调用才进入到运行状态。

### 6. 线程优先级
现代操作系统基本采用时分的形式调度运行的程序，操作系统会分出一个个时间片，线程分配到若干时间片，当线程的时间片用完了就会发生线程调度，并等待着下次的分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而线程优先级就是决定线程需要多或少分配一些处理器资源的线程属性

在Java线程中，通过一个整形成员变量`priority` 来控制优先级，优先级的范围从1~10，在线程构建的时候可以通过`setPriority(int) `方法来修改优先级，**默认优先级是5**，优先级高的线程分配时间片的数量要多于优先级低的线程。**设置线程优先级时，针对频繁阻塞（休眠或者I/O操作）的线程需要设置较高优先级，而偏重计算（需要较多CPU时间或者偏运算）的线程则设置较低的优先级，确保处理器不会被独占**。在不同的JVM以及操作系统上，线程规划会存在差异，有些操作系统甚至**会忽略对线程优先级的设定**

> 线程优先级不能作为程序正确性的依赖，因为操作系统可以完全不用理会java程序对于优先级的设定

### 7. 守护线程
守护(Daemon)线程是一种支持型线程，因为它主要被用作程序中后台调度以及支持性工作。这意味着，当一个Java虚拟机不存在非Daemon线程的时候，Java虚拟机将会推出。可以通过用`Thread.setDaemon(true)`将线程设置为Daemon线程。

> Daemon 属性需要在启动线程之前设置，不能在启动线程之后设置。

Daemon 线程被用作完成支持性工作，但是在Java虚拟机退出时Daemon线程中的**finally块并不一定会执行**,不能依靠finally 块中的内容来确保执行关闭或清理资源的逻辑


### 8. 线程的启动和终止
#### 8.1 启动线程
调用start()方法可启动线程。其含义为：当前线程（parent）同步告知JVM，只要线程规划器空闲，应立即启动调用start()方法的线程。

<font color=red><b>为什么启动线程调用start()而不是run()方法？</b></font>
1. start()用来启动一个线程，当调用start()方法时，系统才会开启一个线程，通过Thead类中start()方法来启动的线程处于就绪状态（可运行状态），**此时并没有运行**，一旦得到CPU时间片，就自动开始执行run()方法。此时不需要等待run()方法执行完也可以继续执行下面的代码，所以也由此看出run()方法并没有实现多线程。 
2. run()方法是在本线程里的，只是线程里的一个函数,而不是多线程的。如果直接调用run(),其实就相当于是调用了一个普通函数而已，直接待用run()方法必须等待run()方法执行完毕才能执行下面的代码，所以执行路径还是只有一条，根本就没有线程的特征，所以在多线程执行时要使用start()方法而不是run()方法。
3. 把需要处理的代码放到run()方法中，start()方法启动线程将自动调用run()方法，这个由java的内存机制规定的。并且run()方法必需是public访问权限，返回值类型为void。
4. 当程序调用start方法一个新线程将会被创建，并且在run方法中的代码将会在新线程上运行

#### 8.2 中断线程
##### 8.2.1 过时的suspend()、resume()、stop()
就是暂停，恢复，停止。
- `suspend()`方法在调用后，**线程不会释放已经占有的资源，而是占用着资源进入睡眠状态**，这样容易引发死锁问题。
- `stop()`方法在终止一个线程时并**不会保证线程的资源能够正常释放**。通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定的状态下。

##### 8.2.2 理解interrupt
Java没有提供一种安全、直接的方法来停止某个线程（stop()太暴力），而是提供了中断机制。中断机制是一种协作机制，也就是说**通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理**。  
其他线程通过调用该线程的`interrupt()`方法对其进行中断操作。只是给它设置了标识。这一方法实际完成的是，给此线程发出一个中断信号。有个例子举个蛮好，就像父母叮嘱出门在外的子女要注意身体一样，父母说了，但是子女是否注意身体、如何注意身体，还是要看自己。

中断机制也是一样的，每个线程对象里都有一个标识位表示是否有中断请求（当然JDK的源码是看不到这个标识位的，是虚拟机线程实现层面的），代表着是否有中断请求。

下面是一个“不听话的子女”的例子：

```java
public class Test {
    public static void main(String[] args) {
        Thread thread = new Thread() {
            public void run() {
                System.out.println("线程启动了");
                //对于这种情况，即使线程调用了intentrupt()方法并且isInterrupted()，但线程还是会继续运行，根本停不下来！   
                while (true) { 
                    System.out.println(isInterrupted());//调用interrupt之后为true
                }
            }
        };
        thread.start();
        //注意，此方法不会中断一个正在运行的线程，它的作用是：在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态
        thread.interrupt();
        while (true) {
            System.out.println("是否isInterrupted：" + thread.isInterrupted());//true
        }
    }
}
```
很明显，上面并不会中断里面的线程，这个线程就是个死循环，才不管你给它的中断信号。
而`interrupt()`的正确用法是**给受阻塞的线程发出一个中断信号，这样受阻线程就得以退出阻塞的状态**。 更确切的说，如果线程被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞，此时调用该线程的interrupt()方法，那么该线程将抛出一个 InterruptedException中断异常（该线程必须事先预备好处理此异常），从而提早地终结被阻塞状态。如果线程没有被阻塞，这时调用 interrupt()将不起作用，直到执行到wait(),sleep(),join()时,才马上会抛出 InterruptedException。

正确使用方法：

1. **使用 interrupt() + InterruptedException来中断线程**  
线程处于阻塞状态，如Thread.sleep、wait、IO阻塞等情况时，调用interrupt方法后，sleep等方法将会抛出一个InterruptedException：
```java
public static void main(String[] args) {
    Thread thread = new Thread() {
        public void run() {
            System.out.println("线程启动了");
            try {
                Thread.sleep(1000 * 100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程结束了");
        }
    };
    thread.start();

    try {
        Thread.sleep(1000 * 5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    thread.interrupt();//作用是：在线程阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态
}
```
2. **使用 interrupt() + isInterrupted()来中断线程**  
`this.interrupted()`:测试当前线程是否已经中断（静态方法）。如果连续调用该方法，则第二次调用将返回false。在api文档中说明interrupted()方法具有清除状态的功能。执行后具有将状态标识清除为false的功能。  
`this.isInterrupted()`:测试线程是否已经中断，但是不能清除状态标识。
```java
// 与上面不同就是判断条件变了
while (!isInterrupted()) {
    System.out.println(isInterrupted());//调用 interrupt 之后为true
}   
```

##### 8.2.3 安全的终止线程
根据上面的思想，其实我们可以自定义一个标识位，来判断是否终止线程。

```java
public class test1 {

    public static volatile boolean exit =false;  //退出标志
    
    public static void main(String[] args) {
        new Thread() {
            public void run() {
                System.out.println("线程启动了");
                while (!exit) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("线程结束了");
            }
        }.start();
        
        try {
            Thread.sleep(1000 * 5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        exit = true;//5秒后更改退出标志的值,没有这段代码，线程就一直不能停止
    }
}
```
或者是线程内部定义私有变量，然后公开一个方法，改变其状态，外部可以通过此方法，改变状态，从而停止该线程。（判断条件一致）

### 9. 并发的三大性质
#### 9.1 原子性
原子性是指一个操作是不可中断的，要么全部执行成功要么全部执行失败，有着“同生共死”的感觉。及时在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程所干扰。

```java
int a = 10; //原子操作
a++; // 其余都不是
int b=a;
a = a+1; 
```
java内存模型中定义了8中操作都是原子的，不可再分的。

1. lock(锁定)：作用于主内存中的变量，它把一个变量标识为一个线程独占的状态；
2. unlock(解锁):作用于主内存中的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
3. read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便后面的load动作使用；
4. load（载入）：作用于工作内存中的变量，它把read操作从主内存中得到的变量值放入工作内存中的变量副本
5. use（使用）：作用于工作内存中的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作
6. assign（赋值）：作用于工作内存中的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作；
7. store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送给主内存中以便随后的write操作使用；
8. write（操作）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

> 大致认为基本数据类型的访问读写具备原子性（例外就是long和double的非原子性协定）


#### 9.2 有序性
有序性：即程序执行的顺序按照代码的先后顺序执行。  
指令重排序不会影响单个线程的执行，但是会影响到线程并发执行的正确性。多线程需要考虑重排序带来的影响。

#### 9.4 可见性
可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。典型的就是volatile



> 参考
>
> [Java线程的6种状态及切换(透彻讲解)](https://blog.csdn.net/qq_22771739/article/details/82529874?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)  
> [Java多线程中断机制三种方法及示例](https://www.jb51.net/article/128207.htm)   
> [java 中断线程的几种方式 interrupt()](https://www.cnblogs.com/myseries/p/10918819.html)  
> [三大性质总结：原子性，有序性，可见性](https://www.jianshu.com/p/cf57726e77f2)



---



## 1.2.2 线程通信和同步
> 基于《Java并发编程的艺术》

### 1. 通信和同步
在并发编程中，需要处理两个关键问题：线程之间如何通信及线程之间如何同步（这里的线程是指并发执行的活动实体）。通信是指线程之间以何种机制来交换信息。在命令式编程中，线程之间的通信机制有两种：**共享内存和消息传递**。

- **在共享内存的并发模型里**，线程之间共享程序的公共状态，线程之间通过写-读内存中的公共状态来隐式进行通信。
- **在消息传递的并发模型里**，线程之间没有公共状态，线程之间必须通过明确的发送消息来显式进行通信。

**同步**是指程序用于控制不同线程之间操作发生相对顺序的机制。
- 在共享内存并发模型里，**同步是显式进行的**。程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。
- 在消息传递的并发模型里，由于消息的发送必须在消息的接收之前，因此**同步是隐式进行的**。

**Java的并发采用的是共享内存模型**，Java线程之间的通信总是隐式进行，整个通信过程对程序员完全透明。如果编写多线程程序的java程序员不理解隐式进行的线程之间通信的工作机制，很可能会遇到各种奇怪的内存可见性问题。

### 2. Java内存模型
这个JVM中也讲到了。

Java内存模型的抽象在java中，所有实例域、静态域和数组元素存储在堆内存中，堆内存在线程之间共享（本文使用“共享变量”这个术语代指实例域，静态域和数组元素）。局部变量（Local variables），方法定义参数（java语言规范称之为formal method parameters）和异常处理器参数（exception handler parameters）不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。
Java线程之间的通信由**Java内存模型**（本文简称为JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，**JMM定义了线程和主内存之间的抽象关系**：线程之间的**共享变量存储在主内存**（main memory）中，每个线程都有一个私有的本地内存（local memory），**本地内存中存储了该线程以读/写共享变量的副本**。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。Java内存模型的抽象示意图如下：

![Java内存模型](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200730/201517wr21djsdf25qcdrn.png)


从上图来看，线程A与线程B之间如要通信的话，必须要经历下面2个步骤：
1. 线程A把本地内存A中更新过的共享变量刷新到主内存中去。
2. 线程B到主内存中去读取线程A之前已更新过的共享变量。  
下面通过示意图来说明这两个步骤：
![Java内存模型](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200730/201517f5mzxmms9dxoimmu.png)

这种通过主内存的通信当然可能造成线程安全问题了。


### 3. happen-before原则

1. **程序顺序规则**：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
2. **监视器锁规则**：对一个锁的解锁，happens-before于随后对这个锁的加锁。
3. **volatile变量规则**：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
4. **传递性**：如果A happens-before B，且B happens-before C，那么A happens-before C。
5. start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
6. Join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
7. 程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
8. 对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。

与程序员密切相关的规则是前4个。
> 需要注意的是两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！**happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见**，且前一个操作按顺序排在第二个操作之前。其实就是可以被**重排序**。


### 4. 重排序
**重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。**

简单的例子：

```java
int a = 1;
int b = 3;
int c = a + b
```
对于上面的代码，a,b变量的赋值就有可能被重排序。至于第三步不可能被重排序，因为它依赖于前两步。重排序保证（单线程）程序的执行结果不能被改变（as-if-serial语义）。

但是**多线程下重排序会改变程序的执行结果**：

```java
class ReorderExample {
	int a = 0;
	boolean flag = false;
 
	public void writer() {
		a = 1;          // 1
		flag = true;    // 2
	}
 
	public void reader() {
		if (flag) {            // 3
			int i = a * a; // 4
		}
	}
}
```
flag变量是个标记，用来标识变量a是否已被写入。这里假设有两个线程A和B，A首先执行writer()方法，随后B线程接着执行reader()方法。线程B在执行操作4时，能否看到线程A在操作1对共享变量a的写入？

答案是：不一定能看到。  
a可能还是0，也可能赋值为1


#### 4.1 重排序的类型
在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3种类型。

1. **编译器优化的重排序**，编译器在不改变单线程程序语义的前提下，可以重新安排语义。
2. **指令级并行的重排序**。现代处理器采用了指令级并行技术（Instruction-Level Parallelism，ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
3. **内存系统的重排序**。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

从Java源代码到最终实际执行的指令序列，会分别经历上面3种重排序。

对于编译器的重排序，JMM的编译器重排序规则会禁止特定类型的编译器重排序（不是所有的编译器重排序都要禁止）。

对于处理器的重排序，JMM的处理器重排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障来禁止特定类型的处理器重排序。

#### 4.2 as-if-serial语义

as-if-serial语义的意思是：**不管怎么重排序，（单线程）程序的执行结果不能被改变**。编译器、runtime和处理器都必须遵守as-if-serial语义。

为了遵守as-if-serial语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可以被编译器和处理器重排序。

为了具体说明，请看下面计算圆面积的代码示例。

```java
double pi = 3.14;            // A
double r = 1.0;                // B
double area = pi * r * r;    // C
```
因为上面A和C以及B和C都存在数据依赖关系，但是A和B不存在数据以来关系。所以A和B的执行实际上可以发生重排序。

这其实给大家都创建了一个幻觉一样：**单线程程序时按程序的顺序来执行的**。as-if-serial语义使单线程程序员无需担心重排序会干扰他们，也无需担心内存可见性问题。(这其实就是上面说的**编译器重排序**)

为保证as-if-serial语义，**Java异常处理机制也会为重排序做一些特殊处理**。例如在下面的代码中，y = 0 / 0可能会被重排序在x = 2之前执行，为了保证最终不致于输出x = 1的错误结果，JIT在重排序时会在catch语句中插入错误代偿代码，将x赋值为2，将程序恢复到发生异常时应有的状态。这种做法的确将异常捕捉的逻辑变得复杂了，但是JIT的优化的原则是，尽力优化正常运行下的代码逻辑，哪怕以catch块逻辑变得复杂为代价，毕竟，进入catch块内是一种“异常”情况的表现。

```java
public class Reordering {
    public static void main(String[] args) {
        int x, y;
        x = 1;
        try {
            x = 2;
            y = 0 / 0;    
        } catch (Exception e) {
        } finally {
            System.out.println("x = " + x);
        }
    }
}
```
#### 4.3 禁止重排序

1. **volatile**禁止重排序，通过插入内存屏障，这个后面再说。
2. **synchronized**：也是使用了内存屏障
3. **final**  
对于final域，编译器和处理器要遵守两个重排序规则。
    - 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。（防止拿到对象时，final域还未赋值）
    - 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。




### 5. 内存屏障
内存屏障这个东西就非常底层了，一般不会问到，除非变态的面试官会说这个。不过还是了解一下吧，像volatile的实现就是加各种内存屏障，如果问道volatile，再把这个说出来，就显得很pro。

为了保证内存可见性，java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为下列四类：

屏障类型 | 指令示例 | 说明    
--- | --- | --- |
LoadLoad Barriers   |  Load1; LoadLoad; Load2       |确保Load1数据的装载，之前于Load2及所有后续装载指令的装载。
StoreStore Barriers |  Store1; StoreStore; Store2   |确保Store1数据对其他处理器可见（刷新到内存），之前于Store2及所有后续存储指令的存储。
LoadStore Barriers  |  Load1; LoadStore; Store2     |确保Load1数据装载，之前于Store2及所有后续的存储指令刷新到内存。
StoreLoad Barriers  |  Store1; StoreLoad; Load2     |确保Store1数据对其他处理器变得可见（指刷新到内存），之前于Load2及所有后续装载指令的装载。StoreLoad Barriers会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令。

**StoreLoad Barriers是一个“全能型”的屏障**，它同时具有其他三个屏障的效果。现代的多处理器大都支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（buffer fully flush）。

> 前面算是基础了，现在开始讲解线程通信同步方法

### 6. volatile

#### 6.1 volatile能保证有序性，可见性，但不能保证原子性。

**可见性**
- 当写一个volatile变量时，JMM会把该线程本地内存中的变量强制刷新到主内存中去；
- **这个写会操作会导致其他线程中的缓存无效**。

**有序性**  
禁止指令重排，在编译时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎是不可能的，为此，JMM采取了保守策略：

- 在每个volatile**写操作的前面**插入一个**StoreStore**屏障，禁止上面的普通写和下面的volatile写重排序；；
- 在每个volatile**写操作的后面**插入一个**StoreLoad**屏障，防止上面的volatile写与下面可能有的volatile读/写重排序；
- 在每个volatile**读操作的后面**插入一个**LoadLoad**屏障，禁止下面所有的普通读操作和上面的volatile读重排序；
- 在每个volatile**读操作的后面**插入一个**LoadStore**屏障，禁止下面所有的普通写操作和上面的volatile读重排序。

![volatile写](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/2615789-a31dbae587e8a946.png)
![volatile读](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/2615789-a31dbae587e8a946.png)

总结：
- 当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后
面的操作可见；在其后面的操作肯定还没有进行；
- 在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放
到其前面执行。

**即执行到volatile变量时，其前面的所有语句都执行完，后面所有语句都未执行。且前面语句的结果对volatile变
量及其后面语句可见**。

#### 6.2 为什么不建议使用volatile
并发专家建议我们远离volatile是有道理的，这里再总结一下：

- volatile是在synchronized性能低下的时候提出的。如今synchronized的效率已经大幅提升，所以volatile存在的意义不大。
- 如今非volatile的共享变量，在访问不是超级频繁的情况下，已经和volatile修饰的变量有同样的效果了。
- volatile不能保证原子性，这点是大家没太搞清楚的，所以很容易出错。


#### 6.3 volatile在单例模式下的应用

```java
public class SingletonTest {
    private volatile static SingletonTest instance = null;
    private SingletonTest() { }
    public static SingletonTest getInstance() {
        if(instance == null) {
            synchronized (SingletonTest.class){
                if(instance == null) {
                    instance = new SingletonTest();  //非原子操作
                }
            }
        }
        return instance;
    }
}
```
instance = new SingletonTest();可分解为：

```java
memory =allocate(); //1 分配对象的内存空间
ctorInstance(memory); //2 初始化对象
instance =memory; //3 设置instance指向刚分配的内存地址
```
操作2依赖1，但是操作3不依赖2，所以有可能出现1,3,2的顺序，当出现这种顺序的时候，虽然instance不为空，但是对象也有可能没有正确初始化，会出错。**设置为volatile，将会禁止2，3之间的重排序**。


### 7. synchronized
在多线程并发编程中 synchronized 一直是元老级的角色，很多人都会直呼它为重量级锁。但是，随着 Java SE 1.6 对 synchronized 进行了各种优化之后，有些情况下synchronized 并没有那么重了。

> synchronized 怎么使用就不说了。

synchronized 实现同步的基础：Java中的每一个对象都可以作为锁。具体的表现形式有以下三种：

- 对于普通同步方法，锁是当前实例对象
- 对于静态同步方法，锁是当前类的Class对象
- 对于同步方法块，锁是 synchronized 括号里配置的对象

当一个线程试图访问同步代码块时，必须先获取到锁，退出或抛出异常时必须释放锁。

如果你对使用了synchronized反编译，就能了解到jvm是使用了什么指令：
- 对于同步方法，是依靠方法修饰符上的`ACC_SYNCHRONIZED`来完成的
- 对于同步块的实现使用了`monitorenter`和`monitorexit`指令

```java
public class Synchronized {
	public static void main(String[] args) {
		// 对Synchronized Class对象进行加锁
		synchronized (Synchronized.class) {
		}// 静态同步方法，对Synchronized Class对象进行加锁
		m();
	}
	public static synchronized void m() {
	}
}
```

```java
public static void main(java.lang.String[]);
// 方法修饰符，表示：public staticflags: ACC_PUBLIC, ACC_STATIC
Code:
	stack=2, locals=1, args_size=1
	0: ldc #1 // class com/murdock/books/multithread/book/Synchronized
	2: dup
	3: monitorenter // monitorenter：监视器进入，获取锁
	4: monitorexit // monitorexit：监视器退出，释放锁
	5: invokestatic #16 // Method m:()V
	8: return
public static synchronized void m();
// 方法修饰符，表示： public static synchronized
flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
Code:
	stack=0, locals=0, args_size=0
	0: return
```
无论采用哪种方式，其本质是对一个对象的监视器（monitor）进行获取，而这个获取过程是排他的，也就是同一时刻只能有一个线程获取到由synchronized所保护对象的监视器。

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块或者同步方法，而没有获取到监视器（执行该方法）的线程将会被阻塞在同步块和同步方法的入口处，进入BLOCKED状态。

> synchronized用的锁是存在Java对象头里的。  
> Java SE 1.6引入了”偏向锁“，”轻量级锁“，提升synchronized性能，这些在后面详细讲”锁“

<font color=red><b>volatile 与 synchronized的区别？</b></font> 

区别 | volatile  |	synchronized
--- | --- | ---
修饰 |	只能用于修饰变量 	| 可以用于修饰方法、代码块
线程阻塞  |	不会发生线程阻塞  |	会发生阻塞
原子性 |	不能保证变量的原子性 	| 可以保证变量原子性
可见性 |	可以保证变量在线程之间访问资源的可见性  |	可以间接保证可见性，因为它会将私有内存中和公共内存中的数据做同步
同步性 |	能保证变量在私有内存和主内存间的同步 	| synchronize是多线程之间访问资源的同步性


### 8. wait/notify

#### 8.1 等待/通知机制
如果没有wait/notify，简单做法是让消费者线程不断地循环检查变量是否符合预期，如下面代码所示：

```
while (value != desire) {
    Thread.sleep(1000);
}
doSomething();
```
这段伪代码在条件不满足时就睡眠一段时间，这样做的目的是防止过快的“无效”尝试，这种方式看似能够解实现所需的功能，但是却存在如下问题。
1. **难以确保及时性**。在睡眠时，基本不消耗处理器资源，但是如果睡得过久，就不能及时发现条件已经变化，也就是及时性难以保证。
2. **难以降低开销**。如果降低睡眠的时间，比如休眠1毫秒，这样消费者能更加迅速地发现条件变化，但是却可能消耗更多的处理器资源，造成了无端的浪费。
以上两个问题，看似矛盾难以调和，但是Java通过内置的等待/通知机制能够很好地解决这个矛盾并实现所需的功能。
等待/通知的相关方法是任意Java对象都具备的，因为这些方法被定义在所有对象的超类java.lang.Object上，方法和描述如下表
![方法](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/20160719155748448.jpg)

**调用wait()、notify()以及notifyAll()时需要注意的细节**:
1. 使用wait()、notify()和notifyAll()时需要**先对调用对象加锁**。
2. 调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程**放置到对象的等待队列**。
3. notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，**需要调用notify()或notifAll()的线程释放锁之后，等待线程才有机会从wait()返回**。
4. notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED。
5. 从wait()方法返回的前提是获得了调用对象的锁。

![运行过程](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/20160719161728210.jpg)

WaitThread（自定义的一个等待的线程）首先获取了对象的锁，然后调用对象的wait()方法，从而放弃了锁并进入了对象的等待队列WaitQueue中，进入等待状态。由于WaitThread释放了对象的锁，NotifyThread（自定义的一个唤醒的线程）随后获取了对象的锁，并调用对象的notify()方法，将WaitThread从WaitQueue移到SynchronizedQueue中，此时WaitThread的状态变为阻塞状态。NotifyThread释放了锁之后，WaitThread再次获取到锁并从wait()方法返回继续执行。

#### 8.2 等待/通知的经典范式

该范式分为两个部分，分别针对等待方（消费者）和通知方（生产者）。

**等待方（消费者）遵循的原则**：
1. 获取对象的锁。
2. 如果条件不满足，那么调用对象的wait()方法，被通知后仍然要检查条件。
3. 条件满足则执行对应的逻辑。

通知方（生产者）遵循如下原则：
1. 获取对象的锁
2. 改变条件
3. 通知所有等待在对象上的线程

<font color=red><b>为什么wait/notify方法定义在Object类中？</b></font> 
1. 这些方法存在于同步中；
2. 使用这些方法必须标识同步所属的锁；
3. 锁可以是任意对象，所以任意对象调用方法一定定义在Object类中。 

<font color=red><b>wait(),sleep()区别？</b></font> 

wait():释放资源，释放锁  
sleep():释放资源，不释放锁

> Condition也提供了类似的等待/通知机制，这个放到后面介绍Lock时说。


### 9. Thread.join()
如果一个线程A执行了`thread.join()`语句，其含义是：当前线程A等待thread线程终止之后才从`thread.join()`返回。线程Thread除了提供`join()`方法之外，还提供了`join(long millis)`和`join(longmillis,int nanos)`两个具备超时特性的方法。这两个超时方法表示，如果线程thread在给定的超时时间里没有终止，那么将会从该超时方法中返回。

> join() 和 sleep() 一样，都可以被中断（被中断时，会抛出 InterrupptedException 异常）；不同的是，join() 内部调用了 wait()，会出让锁，而 sleep() 会一直保持锁。

### 10. ThreadLocal
#### 10.1 简介
ThreadLocal提供了线程的局部变量，每个线程都可以通过set()和get()来对这个局部变量进行操作，但不会和其他线程的局部变量进行冲突，实现了线程的数据隔离。简要言之：往ThreadLocal中填充的变量属于当前线程，该变量对其他线程而言是隔离的。

**ThreadLocal设计的目的就是为了能够在当前线程中有属于自己的变量，并不是为了解决并发或者共享变量的问题**。

#### 10.2 ThreadLocal简单使用

```java
 public class ThreadLocalTest {
 4 
 5     static ThreadLocal<String> localVar = new ThreadLocal<>();
 6 
 7     static void print(String str) {
 8         //打印当前线程中本地内存中本地变量的值
 9         System.out.println(str + " :" + localVar.get());
10         //清除本地内存中的本地变量
11         localVar.remove();
12     }
13 
14     public static void main(String[] args) {
15         Thread t1  = new Thread(new Runnable() {
16             @Override
17             public void run() {
18                 //设置线程1中本地变量的值
19                 localVar.set("localVar1");
20                 //调用打印方法
21                 print("thread1");
22                 //打印本地变量
23                 System.out.println("after remove : " + localVar.get());
24             }
25         });
26 
27         Thread t2  = new Thread(new Runnable() {
28             @Override
29             public void run() {
30                 //设置线程1中本地变量的值
31                 localVar.set("localVar2");
32                 //调用打印方法
33                 print("thread2");
34                 //打印本地变量
35                 System.out.println("after remove : " + localVar.get());
36             }
37         });
38 
39         t1.start();
40         t2.start();
41     }
42 }
```

#### 10.3 ThreadLocal实现的原理

ThreadLocalMap是ThreadLocal的一个内部类。用Entry类来进行存储

我们的值都是存储到这个Map上的，key是当前ThreadLocal对象（也就是”this“）
```java
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        //....
}
```
如果该Map不存在，则初始化一个：

```java
 void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

还有一点需要注意的是：**ThreadLocalMap是在ThreadLocal中使用内部类来编写的，但对象的引用是在Thread中**
```java
// Thread维护了ThreadLocalMap变量
ThreadLocal.ThreadLocalMap threadLocals = null
```
现在再来看set()/get()就很清晰了：

```java
public void set(T value) {
    //取得当前线程，并获取它的threadLocals
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);//没有则创建
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

1. 每个Thread维护着一个ThreadLocalMap的引用
2. ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储
3. 调用ThreadLocal的set()方法时，实际上就是往ThreadLocalMap设置值，key是ThreadLocal对象，值是传递进来的对象
4. 调用ThreadLocal的get()方法时，实际上就是往ThreadLocalMap获取值，key是ThreadLocal对象
5. **ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取value**。

#### 10.4 ThreadLocal内存泄露
![ThreadLocal](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/91ef76c6a7efce1b563edc5501a900dbb58f6512.jpeg)

ThreadLocalMap中的Entry的key使用的是ThreadLocal对象的弱引用，再没有其他地方对ThreadLoca依赖，ThreadLocalMap中的ThreadLocal对象就会被回收掉，但是对应value的不会被回收，这个时候Map中就可能存在key为null但是value不为null的项，由于**ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏**。这需要实际的时候使用完毕及时**调用remove方法**避免内存泄漏。


#### 10.5 ThreadLocal的应用

##### 1. Spring实现事务隔离级别
Spring采用Threadlocal的方式，来**保证单个线程中的数据库操作使用的是同一个数据库连接**，同时，采用这种方式可以使业务层使用事务时不需要感知并管理connection对象，通过传播级别，巧妙地管理多个事务配置之间的切换，挂起和恢复。Spring框架里面就是用的ThreadLocal来实现这种隔离，主要是在TransactionSynchronizationManager这个类里面
##### 2. 避免一些参数传递
避免一些参数的传递的理解可以参考一下Cookie和Session：

- 每当我访问一个页面的时候，浏览器都会帮我们从硬盘中找到对应的Cookie发送过去。
- 浏览器是十分聪明的，不会发送别的网站的Cookie过去，只带当前网站发布过来的Cookie过去

项目中存在一个线程经常遇到横跨若干方法调用，需要传递的对象，也就是上下文（Context），它是一种状态，经常就是是用户身份、任务信息等，就会存在过渡传参的问题。  
也就是方法不需要再设置很多参数传递，而是方法内部直接get()

```java
public void consult(IdCard idCard,StudentCard studentCard,HourseCard hourseCard){
    ...
}
// 改为以下这种
public void consult(){
    threadLocal.get();
}
```
##### 3. 分布式追踪的上下⽂的存储和传递
ThreadLocal + spanId，当前节点的spanId作为下
个节点的⽗spanId

##### 4. dubbo的RpcContext隐式传参
RpcContext是一个ThreadLocal的临时状态记录器，当接收到RPC请求，或发起RPC请求时，RpcContext的状态都会变化。比如A调用B，B再调用C，则B机器上，在B调用C之前，RpcContext记录的是A调用B的信息，在B调用C之后，RpcContext记录的是B调用C。  
线程池创建的ThreadLocal要在
finally中⼿动remove，不然会有内存泄漏的问题

##### 5. 读写锁中
读锁中每个线程各自获取读锁的次数只能保存在ThreadLoacal中。

> 参考
>
> [[转]内存模型之重排序](https://www.jianshu.com/p/6b3615e08882)
> [让你彻底理解volatile](https://www.jianshu.com/p/157279e6efdb)
> [Java中的ThreadLocal详解](https://www.cnblogs.com/fsmly/p/11020641.html)
> [什么是ThreadLocal](https://www.zhihu.com/question/341005993/answer/793627819)



---



## 1.2.3 Java中的锁
> 基于《Java并发编程的艺术》

### 1. 锁的状态
Java SE1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，所以在Java SE1.6里锁一共有四种状态（针对synchronized），由低到高为：无锁状态，偏向锁状态，轻量级锁状态和重量级锁状态，它会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率，下文会详细分析。

#### 1.1 偏向锁

Hotspot的作者经过以往的研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，**为了让线程获得锁的代价更低而引入了偏向锁**。当一个线程访问同步块并获取锁时，会在**对象头和栈帧中的锁记录里存储锁偏向的线程ID**，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

**偏向锁的撤销**：偏向锁使用了一种**等到竞争出现才释放锁**的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

**关闭偏向锁**：偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟-XX：BiasedLockingStartupDelay = 0。如果你确定自己应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁-XX:-UseBiasedLocking=false，那么默认会进入轻量级锁状态。

#### 1.2 轻量级锁
##### 1. 轻量级锁加锁
线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。**如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁**。

##### 2. 轻量级锁解锁
轻量级解锁时，会使用原子的CAS操作来将Displaced Mark Word替换回到对象头，**如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁**。

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

##### 3. 锁的优缺点对比

锁 | 优点 | 缺点 | 适用场景
---|---|---|---
偏向锁 | 加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距。| 如果线程间存在锁竞争，会带来额外的锁撤销的消耗。| 适用于只有一个线程访问同步块场景。
轻量级锁|竞争的线程不会阻塞，提高了程序的响应速度。| 如果始终得不到锁竞争的线程使用自旋会消耗CPU。| 追求响应时间。同步块执行速度非常快。
重量级锁 | 线程竞争不使用自旋，不会消耗CPU。| 线程阻塞，响应时间缓慢。| 追求吞吐量。同步块执行速度较长。


##### 4. CAS
这里介绍一下CAS操作，CAS 即 compare and swap（比较并交换），是一种有名的无锁算法。即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization

Java 从 JDK1.5 开始支持，java.util.concurrent 包里提供了很多面向并发编程的类，也提供了 CAS 算法的支持，一些以 **Atomic** 为开头的一些原子类都使用 CAS 作为其实现方式。

CAS 中涉及三个要素：

- 需要读写的内存值 V
- 进行比较的值 A
- 拟写入的新值 B

当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

**CAS的缺点**： 

**1.ABA问题**  
CAS 的使用流程通常如下：
1. 首先从地址 V 读取值 A；
2. 根据 A 计算目标值 B；
3. 通过 CAS 以原子的方式将地址 V 中的值从 A 修改为 B。

但是在第1步中读取的值是A，并且在第3步修改成功了，我们就能说它的值在第1步和第3步之间没有被其他线程改变过了吗？

**如果在这段期间它的值曾经被改成了B，后来又被改回为A，那CAS操作就会误认为它从来没有被改变过**。这个漏洞称为CAS操作的“ABA”问题。解决思路是，每次变量更新追加版本号。Java并发包(Java 1.5引入)为了解决这个问题，提供了一个带有标记的原子引用类`AtomicStampedReference`，它可以通过控制变量值的版本来保证CAS的正确性。因此，在使用CAS前要考虑清楚“ABA”问题是否会影响程序并发的正确性(ABA问题一般不会带来造成数据的安全性)，如果需要解决ABA问题，改用传统的互斥同步可能会比原子类更高效。

**2.循环时间长开销很大**  
CAS 通常是配合无限循环一起使用的，如果 CAS 失败，会一直进行尝试。如果 CAS 长时间一直不成功，可能会给 CPU 带来很大的开销。

**3.只能保证一个变量的原子操作**  
当对一个变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个变量操作时，CAS 目前无法直接保证操作的原子性。但是我们可以通过以下两种办法来解决：1）使用互斥锁来保证原子性；2）将多个变量封装成对象，通过 AtomicReference 来保证原子性。
> 还有一个取巧方法：把多个共享变量合并为一个，例如a=1,b=5,变为ab=15。

### 2. Lock接口
锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能发挥功能之多个线程同事访问共享资源（读写锁除外）。在Lock接口出现前，Java程序依靠synchronized关键字实现锁功能，在Java 1.5 后，并发包新增了Lock接口以及相关实现类来实现锁功能。它提供了与synchronized关键字类似的同步功能，只是在**使用时需要显示地获取和释放锁**。虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

```java
Lock lock = new ReentrantLock();
lock.lock();
try{

}finally{
	lock.unlock();
}
```
**在finally块中释放锁**，目的是保证在获取到锁之后，最终能够被释放。

**不要将获取锁的过程写在try块中**，因为如果在获取锁（自定义的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放！

方法 |	描述
--- | ---
void lock() | 	获取锁，调用该方法当前线程将会获取锁，当锁获取后，从该方法返回
void lockInterruptibly() throws InterruptedException | 	可中断地获取锁，和lock方法的不同之处在于该方法会响应中断，即在锁的获取中可以中断当前线程
boolean tryLock() |	尝试非阻塞的获取锁，调用该方法后立刻返回，如果能够获取则返回true，否则返回false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException | 	超时的获取锁，当前线程在以下3种情况下会返回：1.当前线程在超时时间内获得了锁；2.当前线程在超时时间内被中断；3.超时时间结束，返回false
void unlock() 	| 释放锁
Condition new Condition() | 	获取等待通知组件，该组件和当前线程的锁绑定，当前线程只有获得了锁，才能调用该组件的wait()方法，而调用后，当前线程将释放锁

<font color=red><b>synchronized与Lock的比较</b></font> 

方面 | synchronized | Lock
--- | --- | ---
使用 | 隐式锁，使用sync关键字的时候，根本不用写其他的代码，然后程序就能够获取锁和释放锁了。| 显示锁，需要手动的获取和释放锁。如果没有释放锁，就有可能导致出现死锁的现象。
底层 | sync是底层是通过monitorenter进行加锁（底层是通过monitor对象来完成的，通过monitorexit来退出锁 | lock是通过调用对应的API方法来获取锁和释放锁的。(AQS，CAS)
中断 | Sync是不可中断的。除非抛出异常或者正常运行完成 | Lock可以中断的。中断方式：1：调用设置超时方法tryLock(long timeout ,timeUnit unit)2：调用lockInterruptibly()放到代码块中，然后调用interrupt()方法可以中断
公平 | 非公平锁 | 两者都可以，默认是非公平锁。在其构造方法的时候可以传入Boolean值
唤醒 | 要么随机唤醒一个线程；要么是唤醒所有等待的线程 | 分组唤醒需要唤醒的线程，可以精确的唤醒
性能 | 1.6之后性能更高，尚有优化余地 | 加油
方式 | 悲观锁 | 乐观锁

### 3. AbstractQueueSynchronizer 队列同步器
简称AQS，它是实现锁（也可以是任意同步组件）的基础组件，juc下面Lock的实现（ReentrantLock，ReentrantReadWriteLock）以及一些并发工具类（CountDownLatch，Semaphore）就是通过AQS来实现的。

先不深入源码了，这里可参考：[深入理解Java中的AQS](https://www.cnblogs.com/fsmly/p/11274572.html)  
[浅谈Java的AQS](https://www.jianshu.com/p/0f876ead2846)

### 4. ReentrantLock 重入锁
#### 4.1 简介
重入锁ReentrantLock,顾名思义，就是支持重进入的锁，它表示**该锁能够支持一个线程对资源的重复加锁**。除此之外，该锁还**支持获取锁时的公平和非公平性选择**。
Mutex是一个不支持重进入的锁，而**synchronized隐式的支持重进入**，比如synchronized修饰的递归方法，执行线程获取了锁之后仍能连续多次的获取该锁。

ReentrantLock在调用lock()方法时，已经获取到锁的线程，能够再次调用lock()获取锁而不被阻塞。

除此之外，该ReentrantLock锁的还支持获取锁时的公平和非公平性选择。那公平性和非公性是如何体现出来的呢？如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。公平的获取锁，也就是**等待时间最长的线程最优先获取锁**，也可以说锁获取是顺序的。ReentrantLock提供了一个构造函数，能够控制锁是否是公平的。

#### 4.2 实现重进入

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，那要实现重进入需要注意什么问题呢？
1. **线程再次获取锁**。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则允许线程再次成功获取锁。
2. **锁的最终释放**。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。这就要求锁对于线程获取时要进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

思路是在锁状态为0时，表示锁空闲，重新尝试获取锁。如果已经被占用，那么判断当前线程是否为占用锁的线程，如果是那么进行计数，当然在锁的relase过程中会进行递减，保证锁的正常释放。

#### 4.3 公平性与非公平性获取锁的区别
公平锁的tryAcquire方法与nonfairTryAcquire(int acquires)比较，唯一不同的位置为判断条件多了 hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。

而在非公平nonfairTryAcquire(int acquires)方法，当一个线程请求锁时，只要获取了同步状态即成功获取锁。在这个前提下，**刚释放锁的线程再次获取同步状态的几率会非常大**，使得其他线程只能在同步队列中等待。这也就是非公平性锁会可能造成使线程“饥饿”的原因，既然这样，它又为什么被设定成默认的实现呢？因为**非公平性锁的开销更小**。如果把每次不同线程获取到锁定义为1次切换，公平性锁在测试中进行了10次切换，而非公平性锁只有5次切换。

### 5. ReentrantReadWriteLock 读写锁
#### 5.1 简介
之前提到锁（如Mutex，ReentrantLock，synchronized）都是排他锁，而**读写锁在同一时刻可以允许多个读线程访问， 但是在写线程访问时，所有的读线程和其他写线程均被阻塞**。**读写锁维护了一对 锁**，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁 有了很大提升。

- 当读写锁是**写**加锁状态时, 在这个锁被解锁之前, 所有试图对这个锁加锁的线程都会被阻塞.
- 当读写锁在**读**加锁状态时, 所有试图以读模式对它进行加锁的线程都可以得到访问权, 但是如果线程希望以写模式对此锁进行加锁, 它必须直到所有的线程释放锁

在没有读写锁支持的（Java5 之前）时候，如果需要完成上述工作就要使用 Java 的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入 等待状态，只有写操作完成并进行通知之后，所有等待的读操作才能继续执行（写 操作之间依靠 synchronized 关键进行同步），这样做的目的是使读操作能读取到 正确的数据，不会出现脏读。改用读写锁实现上述功能，只需要**在读操作时获取读锁，写操作时获取写锁**即可。当写锁被获取到时，后续（非当前写操作线程） 的读写操作都会被阻塞，写锁释放之后，所有操作继续执行，编程方式相对于使 用等待通知机制的实现方式而言，变得简单明了。 

一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。 在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

#### 5.2 实现
##### 5.2.1 读写状态设计
读写锁同样依赖**自定义同步器**（AQS）来实现同步功能,而读写状态就是其同步器的同步状态。回想ReentrantLock中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。  
![读写锁状态](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/2615789-6af1818bbfa83051.png)
根据状态的划分能得出一个推论：S不等于0时，当写状态(S&0x0000FFFF)等于0时，则读状态（S>>>16）大于0，即读锁已被获取。

##### 5.2.2 写锁获取与释放
写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进人等待状态。

```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
//存在读锁或者当前获取的线程不是已经获取写锁的过程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```
该方法除了重入条件(当前线程为获取了写锁的线程)之外，增加了一个读锁是否存在的判断。**如果存在读锁，则写锁不能被获取**，原因在于：**读写锁要确保写锁的操作对读锁可见**，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

##### 5.2.3 读锁的获取与释放
读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进人等待状态。获取读锁的实现从Java 5到Java 6变得复杂许多，主要原因是新增了一些功能，例如getReadHoldCount()方法，作用是返回当前线程获取读锁的次数。读状态是所有线程获取读锁次数的总和，而每个线程各自获取读锁的次数只能选择保存在**ThreadLocal**中，由线程自身维护，这使获取读锁的实现变得复杂。

##### 5.2.4 锁降级
**锁降级指的是写锁降级成为读锁**。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。**锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。**

**锁降级中读锁的获取是否必要呢**？答案是必要的。主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。

**RentrantReadWriteLock不支持锁升级**（把持读锁、获取写锁，最后释放读锁的过程）。目的也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。

### 6. Condition
#### 6.1 简介
之前说过在synchronized中，可以使用`wait/notify`实现等待通知机制，也可以使用Condition的`await/singal`。不过这两者使用还是有些不同的的。Condition可以唤醒指定的线程。Condition对象依赖于Lock对象（调用Lock对象的newCondition()方法)。Condition的使用方式非常的简单。但是需要注意在调用方法前获取锁。

```java
private Lock lock = new ReentrantLock();

private Condition condition = lock.newCondition();

public void conditionAwait() throws InterruptedException {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock();
    }
}

public void conditionSignal() {
    lock.lock();
    try {
        condition.signal();//condition.signalAll();
    } finally {
        lock.unlock();
    }
}
```
Condition实现生产消费模式：

```java
  1 
  2 import java.util.LinkedList;
  3 import java.util.concurrent.locks.Condition;
  4 import java.util.concurrent.locks.Lock;
  5 import java.util.concurrent.locks.ReentrantLock;
  6 public class BoundedQueue {
  8 
  9     private LinkedList<Object> buffer;    //生产者容器
 10     private int maxSize ;           //容器最大值是多少
 11     private Lock lock;
 12     private Condition fullCondition;
 13     private Condition notFullCondition;
 14     BoundedQueue(int maxSize){
 15         this.maxSize = maxSize;
 16         buffer = new LinkedList<Object>();
 17         lock = new ReentrantLock();
 18         fullCondition = lock.newCondition();
 19         notFullCondition = lock.newCondition();
 20     }
 21 
 22     /**
 23      * 生产者
 24      * @param obj
 25      * @throws InterruptedException
 26      */
 27     public void put(Object obj) throws InterruptedException {
 28         lock.lock();    //获取锁
 29         try {
 30             while (maxSize == buffer.size()){
 31                 notFullCondition.await();       //满了，添加的线程进入等待状态
 32             }
 33             buffer.add(obj);
 34             fullCondition.signal(); //通知
 35         } finally {
 36             lock.unlock();
 37         }
 38     }
 39 
 40     /**
 41      * 消费者
 42      * @return
 43      * @throws InterruptedException
 44      */
 45     public Object get() throws InterruptedException {
 46         Object obj;
 47         lock.lock();
 48         try {
 49             while (buffer.size() == 0){ //队列中没有数据了 线程进入等待状态
 50                 fullCondition.await();
 51             }
 52             obj = buffer.poll();
 53             notFullCondition.signal(); //通知
 54         } finally {
 55             lock.unlock();
 56         }
 57         return obj;
 58     }
 59 
 60 }
```

#### 6.2 原理
Condition接口其下的实现类有ConditionObject，它是同步器AQS的内部类，以下如果没有说明则Condition指的是ConditionObject。
##### 6.2.1 等待队列 
每个Condition对象都包含一个队列(等待队列)。等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。等待队列的基本结构如下所示。
![等待队列](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/648116-20180515071113479-1426477894.png)

等待分为首节点和尾节点。当一个线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列。新增节点就是将尾部节点指向新增的节点。节点引用更新本来就是在获取锁以后的操作(**调用await()方法的线程必定获取了锁的线程**)，所以不需要CAS保证。同时也是线程安全的操作。

##### 6.2.2 等待
当线程调用了await方法以后。线程就作为队列中的一个节点被加入到等待队列中去了。同时会释放锁。当从await方法返回的时候，当前线程一定会获取condition相关联的锁。当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列。
![当前线程加入到等待队列](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/648116-20180515071118116-198589862.png)  
当等待队列中的节点被唤醒的时候，则唤醒节点的线程开始尝试获取同步状态。如果不是通过 其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException异常信息。


##### 6.2.2 通知
调用Condition的signal()方法，将会唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到同步队列中。在调用signal()方法之前必须先判断是否获取到了锁。接着获取等待队列的首节点，将其移动到同步队列并且利用LockSupport唤醒节点中的线程。节点从等待队列移动到同步队列如下图所示：

![节点从等待队列移动到同步队列](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/648116-20180515071122335-2001301461.png)
被唤醒的线程将从await方法中的while循环中退出。随后加入到同步状态的竞争当中去。成功获取到竞争的线程则会返回到await方法之前的状态。

> Condition的signalAll()方法。相当于对等待队列中的每个节点均质性一次signal()方法，效果就是将等待队列中的所有节点全部移动到同步队列中，并唤醒每个节点的线程

### 7. StampedLock
StampedLock类,在JDK1.8时引入,是对读写锁ReentrantReadWriteLock的增强,该类提供了一些功能,优化了读锁、写锁的访问,同时使读写锁之间可以互相转换,更细粒度控制并发。  
相比于普通的ReentranReadWriteLock主要多了一种乐观读的功能

1. 进入悲观读锁前先看下有没有进入写模式（说白了就是有没有已经获取了悲观写锁）
2. 如果其他线程已经获取了悲观写锁，那么就只能老老实实的获取悲观读锁（**这种情况相当于退化成了读写锁**）
3. 如果其他线程没有获取悲观写锁，那么就不用获取悲观读锁了，减少了一次获取悲观读锁的消耗和避免了因为读锁导致写锁阻塞的问题，直接返回读的数据即可（必须再tryOptimisticRead和validate之间获取好数据，否则数据可能会不一致了，试想如果过了validate再获取数据，这时数据可能被修改并且读操作也没有任何保护措施）

详细参考：[Java多线程进阶（十一）—— J.U.C之locks框架：StampedLock](https://segmentfault.com/a/1190000015808032?utm_source=tag-newest)


> 参考
>
> [java中的各种锁详细介绍](https://www.cnblogs.com/jyroy/p/11365935.html)  
> [重入锁 ReentrantLock](https://www.cnblogs.com/ljl150/p/12568820.html)
> [深入理解读写锁ReentrantReadWriteLock](https://www.jianshu.com/p/4a624281235e)  
> [Java并发之Condition](https://www.cnblogs.com/gemine/p/9039012.html)



---



## 1.2.4 Java中的并发工具类
> 基于《Java并发编程的艺术》

Java 1.5 并发包引入许多有用的工具类，如CountDownLatch，CyclicBarrier，Semaphore

### 1. CountDownLatch
#### 1.1 简介
CountDownLatch允许一个或多个线程等待其他线程完成操作。

其实很容易想到之前的`join`可以实现此需求

```java
/**
 * <Description> join用于让当前执行线程等待join线程执行结束。其实现原理是不停检查join线程是否存活，
 * 如果join线程存活则让当前线程永远等待。其中，wait（0）表示永远等待下去，代码片段如下。
 * while (isAlive()) {wait(0);}
 * 直到join线程中止后，线程的this.notifyAll()方法会被调用，调用notifyAll()方法是在JVM里实现的，
 * 所以在JDK里看不到，大家可以查看JVM源码。
 */
public class JoinCountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {
        Thread parser1 = new Thread(new Runnable() {
            @Override
            public void run() {

            }
        });

        Thread parser2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("parser2 finish!");
            }
        });

        parser1.start();
        parser2.start();
        parser1.join();
        parser2.join();
    }
}
```
使用**CountDownLatch**：

```java
public class CountDownLatchTest {
    static CountDownLatch c = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(1);
                c.countDown();
                System.out.println(2);
                c.countDown();
            }
        }).start();
        c.await();
        System.out.println(3);
    }
}
```
CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。  
当我们**调用CountDownLatch的countDown方法时，N就会减1**，**CountDownLatch的await方法会阻塞当前线程，直到N变成零**。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

如果有某个解析sheet的线程处理得比较慢，我们不可能让主线程一直等待，所以可以使用另外一个带指定时间的await方法——await（long time，TimeUnit unit），这个方法等待特定时间后，就会不再阻塞当前线程。join也有类似的方法。

> 注意:计数器必须大于等于0，只是等于0时候，计数器就是零，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。**一个线程调用countDown方法happen-before，另外一个线程调用await方法**。

模拟并发
```java
public class Parallellimit {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newCachedThreadPool();
        CountDownLatch cdl = new CountDownLatch(100);
        for (int i = 0; i < 100; i++) {
            CountRunnable runnable = new CountRunnable(cdl);
            pool.execute(runnable);
        }
    }
}

 class CountRunnable implements Runnable {
    private CountDownLatch countDownLatch;
    public CountRunnable(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }
    @Override
    public void run() {
        try {
            synchronized (countDownLatch) {
                /*** 每次减少一个容量*/
                countDownLatch.countDown();
                System.out.println("thread counts = " + (countDownLatch.getCount()));
            }
            countDownLatch.await();
            System.out.println("concurrency counts = " + (100 - countDownLatch.getCount()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### 2. CyclicBarrier
#### 2.1 简介
CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。  
它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

CyclicBarrier默认的构造方法是CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。示例代码如下所示：
```java
public class CyclcBarrierTest {
	static CyclicBarrier c = new CyclicBarrier(2);
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					c.await();
				} catch (InterruptedException | BrokenBarrierException e) {
					e.printStackTrace();
				}
				System.out.println(1);
			}
		}).start();
		try {
			c.await();
		} catch (InterruptedException | BrokenBarrierException e) {
			e.printStackTrace();
		}
		System.out.println(2);
	}
}
```
因为主线程和子线程的调度是由CPU决定的，两个线程都有可能先执行，所以会产生两种输出，第一种可能输出如下：1、2；第二种可能输出如下：2、1。

如果把new CyclicBarrier(2)修改成new CyclicBarrier(3)，则主线程和子线程会永远等待，因为没有第三个线程执行await方法，即没有第三个线程达到屏障，所以之前到达屏障的两个线程都不会继续执行。

CyclicBarrier还提供了一个更高级的构造函数CyclicBarrier(int parties, Runnable barrierAction)，用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景：

```java
public class CyclcBarrierTest2 {
	static CyclicBarrier c = new CyclicBarrier(2, new A());
	public static void main(String[] args) {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					c.await();
				} catch (InterruptedException | BrokenBarrierException e) {
					e.printStackTrace();
				}
				System.out.println(1);
			}
		}).start();
		try {
			c.await();
		} catch (InterruptedException | BrokenBarrierException e) {
			e.printStackTrace();
		}
		System.out.println(2);
	}
	
	static class A implements Runnable {
		@Override
		public void run() {
			System.out.println(3);
		}
	}
}
```
因为CyclicBarrier设置了拦截线程的数量是2，所以必须等代码中的第一个线程和线程A都执行完之后，才会继续执行主线程，然后输出2，所以代码执行后的输出如下所示：3、1、2。

#### 2.2 应用场景
**CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景**。例如，用一个Excel保存饿了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水，如下所示

```java
public class BankWaterService implements Runnable {
	/*
	 * 创建4个屏障，处理完之后执行当前类的run方法
	 */
	private CyclicBarrier c = new CyclicBarrier(4, this);
	/*
	 * 假设只有4个sheet，所以只启动4个线程
	 */
	private Executor executor = Executors.newFixedThreadPool(4);
	/*
	 * 保存每个sheet计算出的银流结果
	 */
	private ConcurrentHashMap<String, Integer> sheetBankWaterCount = new ConcurrentHashMap<String, Integer>();
	private void count() {
		for (int i = 0; i < 4; i++) {
			executor.execute(new Runnable() {
				@Override
				public void run() {
					// 计算当前sheet的银流数据，计算代码省略
					sheetBankWaterCount.put(Thread.currentThread().getName(), 1);
					// 银流计算完成，插入一个屏障
					try {
						c.await();
					} catch (InterruptedException | BrokenBarrierException e) {
						e.printStackTrace();
					}
				}
			});
		}
	}
	@Override
	public void run() {
		int result = 0;
		// 汇总每个sheet计算出得结果
		for (Entry<String, Integer> sheet : sheetBankWaterCount.entrySet()) {
			result += sheet.getValue();
		}
		// 将结果输出
		sheetBankWaterCount.put("result", result);
		System.out.println(result);
	}
	public static void main(String[] args) {
		BankWaterService bankWaterCount = new BankWaterService();
		bankWaterCount.count();
	}
}
```
#### 2.3 CyclicBarrier 与 CountDownLatch 区别
- **CountDownLatch 是一次性的，CyclicBarrier 是可循环利用的**。所以CyclicBarrier能处理更为复杂的业务场景。例如如果计算错误，可以重置，再计算一次。
- CountDownLatch 参与的线程的职责是不一样的，有的在倒计时，有的在等待倒计时结束。**CyclicBarrier 参与的线程职责是一样的**。

### 3. Semaphore
#### 3.1 简介
**Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源**。  
Semaphore实现控制并发线程数可以抽象为停车场模型，一个固定车位的停车场，当车位满了，便不再允许新的车辆进入；若当前车库驶出多少辆，则就允许进入多少辆。Semaphore做的就是监控车位大小功能。

#### 3.2 应用
Semaphore可以用于做流量控制，特别公用资源有限的应用场景，比如数据库连接。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发的读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有十个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，我们就可以使用Semaphore来做流控

```java
public class SemaphoreTest {
 
	private static final int THREAD_COUNT = 30;
 
	private static ExecutorService threadPool = Executors
			.newFixedThreadPool(THREAD_COUNT);
 
	private static Semaphore s = new Semaphore(10);
 
	public static void main(String[] args) {
		for (int i = 0; i < THREAD_COUNT; i++) {
			threadPool.execute(new Runnable() {
				@Override
				public void run() {
					try {
						s.acquire();
						System.out.println("save data");
						s.release();
					} catch (InterruptedException e) {
					}
				}
			});
		}
 
		threadPool.shutdown();
	}
}
```
在代码中，虽然有30个线程在执行，但是只允许10个并发的执行。Semaphore的构造方法Semaphore(int permits) 接受一个整型的数字，表示可用的许可证数量。Semaphore(10)表示允许10个线程获取许可证，也就是最大并发数是10。Semaphore的用法也很简单，首先线程**使用Semaphore的acquire()获取一个许可证，使用完之后调用release()归还许可证**。还可以用tryAcquire()方法尝试获取许可证。

### 4. Exchanger 
> 用的不多
#### 4.1 简介
**Exchanger(交换者)是一个用于线程间协作的工具类**。它用于线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这2个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当2个线程都到达同步点时，这2个线程就可以交换数据，将本线程生产出来的数据传递给对方。

#### 4.2 简介
Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两个人的数据，并使用交叉规则得出2个交配结果。Exchanger也可用于校对工作，比如，我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看是否录入一致

```java
public class ExchangerTest {

    private static Exchanger<String> exchanger = new Exchanger<String>();

    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                String A = "数据A";
                try {
                    exchanger.exchange(A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                String B = "数据B";
                try {
                    String A = exchanger.exchange("B");
                    System.out.println("A和B的数据是否一致："+A.equals(B)+";A录入的是："+A);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        threadPool.shutdown();
    }

}
```

### 5. CompletableFuture
JDK5 新增了 Future 接口，用于描述一个异步计算的结果。虽然 Future 以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们的异步编程的初衷相违背，轮询的方式又会耗费无谓的 CPU 资源，而且也不能及时地得到计算结果.

**Future接口可以构建异步应用，但依然有其局限性**。它很难直接表述多个Future 结果之间的依赖性。

- 将两个异步计算合并为一个——这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。
- 等待 Future 集合中的所有任务都完成。
- 仅等待 Future集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果。
- 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）。
- 应对 Future 的完成事件（即当 Future 的完成事件发生时会收到通知，并能使用 Future 计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）


Java 8 中, 新增加了一个包含 50 个方法左右的类--CompletableFuture，它提供了非常强大的 Future 的扩展功能，可以帮助我们简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合 CompletableFuture 的方法。

**CompletableFuture实现异步编程示例**：

```java
public class Shop {
    private String name;
    private Random random = new Random();

    public Shop(String name) {
        this.name = name;
    }

    //直接获取价格
    public double getPrice(String product){
        return calculatePrice(product);
    }
    //模拟延迟
    public static void delay(){
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    //模拟获取价格的服务
    private double calculatePrice(String product){
        delay();
        return random.nextDouble() * product.charAt(0) + product.charAt(1);
    }

    //异步获取价格
    public Future<Double> getPriceAsync(String product){
        CompletableFuture<Double> future = new CompletableFuture<>();
        new Thread(() -> {
            double price = calculatePrice(product);
            // 需要需要长时间计算的任务结束并返回结果时,设置Future返回值
            future.complete(price);
        }).start();
        //无需等待还没结束的计算,直接返回Future对象
        return future;
    }
}
```

```java
调用异步接口：
public class Client {
    public static void main(String[] args){
        Shop shop = new Shop("BestShop");
        long start = System.currentTimeMillis();
        Future<Double> future = shop.getPriceAsync("My Favorite");
        long invocationTime = System.currentTimeMillis() - start;
        System.out.println("调用接口时间：" + invocationTime + "毫秒");

        doSomethingElse();

        try {
            double price = future.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        long retrievalTime = System.currentTimeMillis() - start;
        System.out.println("返回价格消耗时间：" + retrievalTime + "毫秒");

    }

    public static void doSomethingElse(){
        System.out.println("做其他的事情。。。");
    }
}
```
如果上述的Shop类提供的方法都是同步阻塞式的，而且你也没法更改，因为他们是服务提供者。

```java
 /**
 * 也就是这个方法不同
 * (阻塞式)通过名称查询价格
 * @param product
 * @return
 */
public double getPrice(String product) {
       return calculatePrice(product);
}
```
**1.采用顺序查询所有商店的方式**

```java
public List<String> findPrice(String product) {
    List<String> list = shops.stream()
            .map(shop ->
                    String.format("%s price is %.2f RMB",
                            shop.getName(),
                            shop.getPrice(product)))

            .collect(toList());

    return list;
}
```
每一个休眠1s，结果自然是耗费4s多。

**2.使用并行流对请求进行并行操作**

```java
public List<String> findPrice2(String product) {
    List<String> list = shops.parallelStream()
            .map(shop ->
                    String.format("%s price is %.2f RMB",
                            shop.getName(),
                            shop.getPrice(product)))

            .collect(toList());

    return list;
}
```
仅仅花费1s多

**3.使用 CompletableFuture 发起异步请求**

```java
public List<String> findPrice3(String product) {
    List<CompletableFuture<String>> futures = shops.stream()
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> String.format("%s price is %.2f RMB",
                            shop.getName(),
                            shop.getPrice(product)))
            )
            .collect(toList());
    List<String> list = futures.stream()
            .map(CompletableFuture::join)
            .collect(toList());


    return list;
}
```
结果花费2s多
CompletableFuture在任务更多时才有优势，但它可以让你对 Execotor (执行器)进行配置,尤其是线程池的大小,让它以更适合应用需求的方式配置.这是并行流API无法提供的。



关于CompletableFuture详解参考：  ****
[java8实战十:CompletableFuture 组合式异步编程](https://blog.csdn.net/itguangit/article/details/78624404)  
[java8新特性（九）：CompletableFuture多线程并发异步编程](https://blog.csdn.net/sunjin9418/article/details/53321818?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~first_rank_v2~rank_v25-1-53321818.nonecase)   
[CompletableFuture 详解](https://www.jianshu.com/p/6f3ee90ab7d3?from=timeline&isappinstalled=0)  
[猫头鹰的深夜翻译：使用JAVA CompletableFuture的20例子](https://segmentfault.com/a/1190000013452165?utm_source=index-hottest)




> 参考
> 
> [countDownLatch](https://www.jianshu.com/p/e233bb37d2e6)



---



## 1.2.5 线程池
> 基于《Java并发编程的艺术》

> 池化技术相⽐⼤家已经屡⻅不鲜了,线程池、数据库连接池、Http 连接池等等都是对这个思想的应
⽤。池化技术的思想主要是为了减少每次获取资源的消耗，提⾼对资源的利⽤率。

### 1. 为何使用线程池

在开发过程中，合理的使用线程池能够带来3个好处：
1. 降低资源消耗。通过重复利⽤已创建的线程降低线程创建和销毁造成的消耗。
2. 提⾼响应速度。当任务到达时，任务可以不需要的等到线程创建就能⽴即执⾏。
3. 提⾼线程的可管理性。线程是稀缺资源，如果⽆限制的创建，不仅会消耗系统资源，还会降低系
统的稳定性，使⽤线程池可以进⾏统⼀的分配，调优和监控。


### 2. ThreadPoolExecutor
阿里规范强制线程池**不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式**，这样
的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

**Executors 返回的线程池对象的弊端如下**： 
1. FixedThreadPool 和 SingleThreadPool:
允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。 
2. CachedThreadPool 和 ScheduledThreadPool:
允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。

这里不会介绍Executors，偏于底层了。

### 2.1 ThreadPoolExecutor构造函数
```java
//五个参数的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}

//六个参数的构造函数-1
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}

//六个参数的构造函数-2
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}

//七个参数的构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler){
    if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;                         
    }
```
这四个构造函数基本差不多，上面三个都是调用的下面的构造方法，只不过传入默认的值。

#### 1. int corePoolSize
corePoolSize，核心线程数，也称基本线程。当线程池创立时，默认是没有创建出线程的。**除非调用了prestartAllCoreThreads()，线程池会提前创建并启动所有核心线程**。当提交一个任务到线程池时，如果线程池中没有达到核心线程数，会创建一个线程来执行任务，**即使其他空闲的核心线程能够执行此任务也会创建线程**。当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中。

#### 2. int maximumPoolSize
线程总数最大值：**线程池允许创建的最大线程数**。上面说到了，如果核心线程满了，会把任务放到队列中，如果这时候队列也满了，就会创建新线程（前提是此时总的线程数量没超过maximumPoolSize）执行任务。值得注意的是，**如果使用了无界队列这个参数就没什么效果**。

#### 3. long keepAliveTime
该线程池中非核心线程闲置超时时长，非核心线程（也称工作线程）空闲后，保持存活的时间。如果任务多，并且执行时间短，可以调大时间，提高线程的利用率。

#### 4. TimeUnit unit
线程活动保持时间的单位，就是上面keepAliveTime的单位。
可选：
- NANOSECONDS ： 1微毫秒 = 1微秒 / 1000
- MICROSECONDS ： 1微秒 = 1毫秒 / 1000
- MILLISECONDS ： 1毫秒 = 1秒 /1000
- SECONDS ： 秒
- MINUTES ： 分
- HOURS ： 小时
- DAYS ： 天

#### 5. BlockingQueue<Runnable> workQueue
任务队列，核心线程满了，也没有空闲线程，新添加任务存在此队列等待处理，如果队列满了，则新建非核心线程执行任务。
- **ArrayBlockingQueue**：基于数组的有界阻塞队列，此队列按照FIFO原则对元素进行排序
- **LinkedBlockingQueue**：基于链表的**无界**阻塞队列（大小为Integer.MAX_VALUE）,按照FIFO原则对元素进行排序。如上面所说，无界队列会导致maximumPoolSize失效。吞吐量通常高于ArrayBlockingQueue， 静态工厂方法`Executors.newFixedThreadPool()`使用了此队列
- **SynchronousQueue**：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常高于LinkedBlockingQueue，静态工厂方法`Executors.newCachedThreadPool()`使用了此队列
- PriorityBlockingQueue：一个具有优先级的无限阻塞队列

#### 6. ThreadFactory threadFactory
对于我们不重要的一个参数，用于设置创建线程的工厂，例如可以使用guava提供ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字：  
`new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build;`

#### 7. RejectedExecutionHandler handler
饱和策略：当线程池和队列都满了，说明线程池处于饱和状态，那么必须采用一种策略处理提交的新任务。默认是AbortPolicy，JDK1.5提供以下四种饱和策略：
- AbortPolicy：直接抛出异常
- CallerRunsPolicy：只用调用者所在线程来运行任务，如果添加到线程池失败，那么主线程会自己去执行该任务，不会等待线程池中的线程去执行
- DiscardOldestPolicy 将最早进入队列的任务（头元素）删掉腾出空间，再尝试加入队列。
- DiscardPolicy 不处理，丢弃掉

**自定义拒绝策略**：只要实现RejectedExecutionHandler接口，并且实现rejectedExecution方法就可以了。

```java
public class MyRejectPolicy implements RejectedExecutionHandler{
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // doSomething
    }
}
```


### 3. 线程池的工作原理
当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？  
![线程池的处理流程](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/20190430102149610.png)

1. **线程池判断核心线程池里的线程是否都在执行任务**。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
2. **线程池判断工作队列是否已经满**。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。
3. **线程池判断线程池的线程是否都处于工作状态**。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

ThreadPoolExecutor执行execute方法思路差不多：  
![ThreadPoolExecutor执行execute](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/20190430103051723.png)

1. 如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）。
2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。
3. 如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。
4. 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。

ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。

上面的流程分析让我们很直观地了解了线程池的工作原理,让我们再通过**源码来看看是如何实现**的,线程池执行任务的方法如下

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //表示 “线程池状态” 和 “线程数” 的整数
    int c = ctl.get();
    // 如果线程数小于基本线程数，则创建线程并把当前任务 command 作为这个线程的第一个任务(firstTask)
    if (workerCountOf(c) < corePoolSize) {
        // 添加任务成功，即结束
        // 执行的结果，会包装到 FutureTask 
        // 返回 false 代表线程池不允许提交任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 如线程数大于等于基本线程(corePoolSize)数或线程创建失败，则将当前任务放到工作队列中。
    // 如果线程池处于 RUNNING ，把这个任务添加到任务队列 workQueue 中
    if (isRunning(c) && workQueue.offer(command)) {
        /* 若任务进入 workQueue，我们是否需要开启新的线程 
        * 线程数在 [0, corePoolSize) 是无条件开启新线程的 
        * 若线程数已经大于等于 corePoolSize，则将任务添加到队列中，然后进到这里 */
        int recheck = ctl.get();
        // 若线程池不处于 RUNNING ，则移除已经入队的这个任务，并且执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 若线程池还是 RUNNING ，且线程数为 0，则开启新的线程
        // 这块代码的真正意图：担心任务提交到队列中了，但是线程都关闭了
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 若 workQueue 满，到该分支
    // 以 maximumPoolSize 为界创建新 worker，
    // 若失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
    else if (!addWorker(command, false))
        // 抛出RejectedExecutionException异常
        reject(command);
}
```

详细可参考[【转载】线程池与五种线程池策略使用与解析](https://www.jianshu.com/p/c8d68f57d06d)

### 4. 常见线程池
#### 4.1.CachedThreadPool()
缓存线程池，含义是池中不保持固定数量的线程，随需创建，最多可以创建Integer.MAX_VALUE个线程（说一句，这个数量已经大大超过目前任何操作系统允许的线程数了），空闲的线程最多保持60秒，多余的任务在SynchronousQueue（一个不存储元素的阻塞队列）中等待。

#### 4.2.FixedThreadPool()
定长线程池：

- 可控制线程最大并发数（同时执行的线程数）
- 超出的线程会在队列中等待

使用了LinkedBlockingQueue（无界阻塞队列）

#### 4.3.ScheduledThreadPool()
定长线程池：

- 支持定时及周期性任务执行。

#### 4.4.SingleThreadExecutor()
单线程化的线程池：

- 有且仅有一个工作线程执行任务
- 所有任务按照指定顺序执行，即遵循队列的入队出队规则

### 5. 向线程池提交任务
可以使用两个方法向线程池提交任务,分别为execute()和submit()方法。
#### 5.1 execute()
用于**提交不需要返回值的任务**,所以**无法判断任务是否被线程池执行成功**.通过以下代码可知execute()方法输入的任务是一个Runnable类的实例.

```java
threadsPool.execute(new Runnable() {
    @Override
    public void run() {
           // TODO Auto-generated method stub
    }
});
```
#### 5.2 submit()
submit()用于提交需要返回值的任务，线程池会返回一个future类型对象,通过此对象可以判断任务是否执行成功，并可通过get()获取返回值,**get()会阻塞当前线程直到任务完成**,而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回,这时候可能任务没有执行完.

```java
Future<Object> future = executor.submit(harReturnValuetask);
    try {
        Object s = future.get();
    } catch (InterruptedException e) {
        // 处理中断异常
    } catch (ExecutionException e) {
        // 处理无法执行任务异常
    } finally {
        // 关闭线程池
        executor.shutdown();
    }
```

### 6. 线程池状态
源码中对5个状态定义：

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
![线程池状态](https://sunsasdoc.oss-cn-hangzhou.aliyuncs.com/image/20200806/08000847-0a9caed4d6914485b2f56048c668251a.jpg)

#### 6.1 RUNNING

**状态说明**：线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理。

**状态切换**：**线程池的初始化状态是RUNNING**。换句话说，线程池被一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0！

#### 6.2 SHUTDOWN
**状态说明**：线程池处在SHUTDOWN状态时，不接收新任务，**但能处理已添加的任务**。

**状态切换**：调用线程池的`shutdown()`接口时，线程池由RUNNING -> SHUTDOWN。

#### 6.3 STOP
**状态说明**：线程池处在STOP状态时，不接收新任务，**不处理已添加的任务，并且会中断正在处理的任务**。 
**状态切换**：调用线程池的`shutdownNow()`接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP。

#### 6.4 TIDYING
整理  
**状态说明**：当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。

**状态切换**：当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 
当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。

#### 6.5 TERMINATED
**状态说明**：**线程池彻底终止**，就变成TERMINATED状态。 

**状态切换**：线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。

### 7. 合理配置线程池
要想合理地配置线程池,就必须首先分析任务特性,可从以下几个角度来分析

- 任务的性质：CPU密集型任务、IO密集型任务和混合型任务
- 任务的优先级：高、中和低
- 任务的执行时间：长、中和短
- 任务的依赖性：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理.

- **CPU密集型任务应配置尽可能小的线程**,如配置N(CPU)+1个线程的线程池
- 由于**I/O密集型任务**线程并不是一直在执行任务,则应配置尽可能多的线程,如2*N(CPU)
- **混合型的任务**,如果可以拆分,将其拆分成一个CPU密集型任务和一个IO密集型任务,只要这两个任务执行的时间相差不是太大,那么分解后执行的吞吐量将高于串行执行的吞吐量.如果这两个任务执行时间相差太大,则没必要进行分解.

可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数.

优先级不同的任务可以使用PriorityBlockingQueue处理.它可以让优先级高
的任务先执行.

> 如果一直有优先级高的任务提交到队列里,那么优先级低的任务可能永远不能执行

执行时间不同的任务可以交给不同规模的线程池来处理,或者可以使用优先级队列,让执行时间短的任务先执行.

**依赖数据库连接池的任务**,因为线程提交SQL后需要等待数据库返回结果,等待的时间越长,则CPU空闲时间就越长,那么**线程数应该设置得越大**,这样才能更好地利用CPU.

**建议使用有界队列**   有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点,比如几千.
假如系统里后台任务线程池的队列和线程池全满了,不断抛出抛弃任务的异常,通过排查发现是数据库出现了问题,导致执行SQL变得非常缓慢,因为后台任务线程池里的任务全是需要向数据库查询和插入数据的,所以导致线程池里的工作线程全部阻塞,任务积压在线程池里.
如果我们设置成无界队列,那么线程池的队列就会越来越多,有可能会撑满内存,导致整个系统不可用,而不只是后台任务出现问题.



> 参考
>
> [线程池，这一篇或许就够了](https://www.jianshu.com/p/210eab345423)  
> [【转载】线程池与五种线程池策略使用与解析](https://www.jianshu.com/p/c8d68f57d06d)


> 关于多线程就介绍这么多了，除了一些比较底层的没讲，例如Java的阻塞队列，AQS具体实现，Fork/join,future等等，这些比较难理解，而且面试中也很冷门，就略过了。这里推荐一位大佬的专栏[透彻理解Java并发编程](https://segmentfault.com/blog/ressmix_multithread),非常底层，如果想深入研究，可以去看看。