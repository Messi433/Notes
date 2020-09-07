### 4.6  Monitor 

#### 4.6.1 Java 对象头

通常我们写的Java对象，在内存中由两部分组成，首先是其对象头，其次是它的成员变量

以 32 位虚拟机为例

普通对象

> Klass Word：指向对象的类型（一个指针找到它的类对象）

```ruby
|--------------------------------------------------------------|
| Object Header (64 bits) |
|------------------------------------|-------------------------|
| Mark Word (32 bits) | Klass Word (32 bits) |
|------------------------------------|-------------------------|
```
数组对象
```ruby
|---------------------------------------------------------------------------------|
| Object Header (96 bits) |
|--------------------------------|-----------------------|------------------------|
| Mark Word(32bits) | Klass Word(32bits) | array length(32bits) |
|--------------------------------|-----------------------|------------------------|
```
其中 Mark Word 结构为

> age：垃圾回收时的分代年龄
>
> biased_lock：是否为偏向锁
>
> 01/00（biased_lock后一位）：加锁状态
>
> Normal：对象正常状态
>
> 当对对象进行相应改变，如施加轻量级锁、重量级锁，GC时，相应的Mark Word Structure会发生改变

```ruby
|-------------------------------------------------------|--------------------|
| Mark Word (32 bits) | State |
|-------------------------------------------------------|--------------------|
| hashcode:25 | age:4 | biased_lock:0 | 01 | Normal |
|-------------------------------------------------------|--------------------|
| thread:23 | epoch:2 | age:4 | biased_lock:1 | 01 | Biased |
|-------------------------------------------------------|--------------------|
| ptr_to_lock_record:30 | 00 | Lightweight Locked |
|-------------------------------------------------------|--------------------|
| ptr_to_heavyweight_monitor:30 | 10 | Heavyweight Locked |
|-------------------------------------------------------|--------------------|
| | 11 | Marked for GC |
|-------------------------------------------------------|--------------------|
```
64 位虚拟机 Mark Word
```ruby
|--------------------------------------------------------------------|--------------------|
| Mark Word (64 bits)                                                |       State        |
|--------------------------------------------------------------------|--------------------|
| unused:25 | hashcode:31 | unused:1 | age:4 | biased_lock:0 | 01    |       Normal       |
|--------------------------------------------------------------------|--------------------|
| thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | 01        |       Biased       |
|--------------------------------------------------------------------|--------------------|
| ptr_to_lock_record:62 | 00 | Lightweight                           |       Locked       |
|--------------------------------------------------------------------|--------------------|
| ptr_to_heavyweight_monitor:62 | 10 |                               | Heavyweight Locked |
|--------------------------------------------------------------------|--------------------|
|                                                            | 11    |    Marked for GC   |
|--------------------------------------------------------------------|--------------------|
```
> 参考资料：https://stackoverflow.com/questions/26357186/what-is-in-java-object-header

#### 4.6.2 Monitor概念

Monitor被翻译为监视器或管程（通常称为“锁”）

每个Java对象都可以关联一个Monitor对象，如果使用synchronized给对象上锁(重量级)之后,该对象头的Mark Word中就被设置指向Monitor对象的指针

##### Monitor结构

> Owner：锁的拥有者，唯一性，当线程尝试获得锁时若有其他线程引用，则无法获得
>
> EntryList：阻塞（等待）队列，其他线程无法获得锁时，则一起进入阻塞队列，但是一旦线程释放锁，它们是竞争获得锁（而不是先来后到）
>
> WaitSet：

**如图1**

obj对象的`MarkWord`结构指向`Monitor`对象，当t2执行到`synchronized`方法时，首先判断临界区代码是否加锁。

如图t2首先判断Monitor Owner是否有线程引用，无则获得锁，执行临界区代码，其他线程t1，t3则进入阻塞队列，等待t2释放锁。

<img src="img\monitor结构2.jpg" alt="monitor结构2" style="zoom:67%;" />

**如图2**

- 刚开始`Monitor`中`Owner`为`null`
- 当`Thread-2` 执行`synchronized(obj)`就会将Monitor的所有者Owner置为`Thread-2`, `Monitor`中只能有一个`Owner`
- 在`Thread-2`上锁的过程中，如果`Thread-3，Thread-4， Thread-5` 也来执行`synchronized(obj)`, 就会进入`EntryList BLOCKED`
- `Thread-2`执行完同步代码块的内容，然后唤醒EntryList中等待的线程来竞争锁，竞争的时是非公平的
- 图中`WaitSet`中的`Thread-0`，`Thread-1` 是之前获得过锁，但条件不满足进入WAITING状态的线程，后面讲`wait-notify`时会分析

<img src="img\monitor结构.jpg" alt="monitor结构" style="zoom:67%;" />

> - synchronized必须是进入**同一个对象**的monitor才有上述的效果
> - 不加synchronized的对象不会关联监视器，不遵从以上规则

##### sychronized原理

代码：

```java
static final object lock = new object();
static int counter = 0;

public static void main(String[] args) {
    synchronized (1ock) {
        counter++ ;
    }
}
```

对应字节码

```java
public static void main(java.lang.String[]);
	descriptor: ([Ljava/lang/String;)V
	flags: ACC_ PUBLIC, ACC_ STATIC
	Code:
		stack=2，1ocals=3, args_ size=1 .
            0: getstatic       #2         // <- lock引用 (synchronized开始)
            3:dup
            4: astore_1                   // 1ock引用 -> slot 1
            5: monitorenter               // 将lock对象MarkWord 置为Monitor 指针
            6: getstatic                  // <-i
            9: iconst_1                   // 准备常数 1
            10: iadd                      // +1
            11: putstatic      #3         // ->i
            14: aload_1                   // <- lock引用
            15: monitorexit               //将lock对象MarkWord 重置，唤醌EntryList
            16: goto           24
                  
            //----------------下面是异常处理时，释放锁的字节码-----------------//
                  
            19: astore_2                  // e->slot2
            20: aload_1                   // <- lock引用
            21: monitorexit               // 将lock对象MarkWord 重置，唤醒EntryList
            22: aload_2                   // <-slot 2 (e)
            23: athrow                    // throw e
            24: return
            Exception table:
            from      to    target type
               6      16    19     any
              19      22    19     any
            LineNumberTable:
```

##### 小故事**

前言：

- synchronized加锁是关联monitor，monitor是由操作系统提供的，成本昂贵，对程序的性能有影响。

- 从 Java6 开始对synchronized获取锁的方式进行了改进

故事角色

- 老王 - JVM
- 小南 - 线程
- 小女 - 线程
- 房间 - 对象
- 房间门上 - 防盗锁 - Monitor
- 房间门上 - 小南书包 - 轻量级锁
- 房间门上 - 刻上小南大名 - 偏向锁
- 批量重刻名 - 一个类的偏向锁撤销到达 20 阈值
- 不能刻名字 - 批量撤销该类对象的偏向锁，设置该类不可偏向

**重量级锁：**小南要使用房间保证计算不被其它人干扰（原子性），最初，他用的是防盗锁，当上下文切换时，锁住门。这样，即使他离开了，别人进不了门，他的工作就是安全的。

但是，很多情况下没人跟他来竞争房间的使用权。小女是要用房间，但使用的时间上是错开的，小南白天用，小女晚上用。每次上锁太麻烦了，有没有更简单的办法呢？

**轻量级锁：**小南和小女商量了一下，约定不锁门了，而是谁用房间，谁把自己的书包挂在门口，但他们的书包样式都一样，因此每次进门前得**翻翻书包（CAS操作）**，看课本是谁的，如果是自己的，那么就可以进门，这样省的上锁解锁了。万一书包不是自己的，那么就在门外等，并通知对方下次用锁门的方式。

后来，小女回老家了，很长一段时间都不会用这个房间。小南每次还是挂书包，翻书包，虽然比锁门省事了，但仍然觉得麻烦。

**偏向锁：**于是，小南干脆在门上刻上了自己的名字：【小南专属房间，其它人勿用】，下次来用房间时，只要名字还在，那么说明没人打扰，还是可以安全地使用房间。如果这期间有其它人要用这个房间，那么由使用者将小南刻的名字擦掉，升级为挂书包的方式。

**批量重刻名：**同学们都放假回老家了，小南就膨胀了，在 20 个房间刻上了自己的名字，想进哪个进哪个。后来他自己放假回老家了，这时小女回来了（她也要用这些房间），结果就是得一个个地擦掉小南刻的名字，升级为挂书包的方式。老王觉得这成本有点高，提出了一种批量重刻名的方法，他让小女不用挂书包了，可以直接在门上刻上自己的名字

后来，刻名的现象越来越频繁，老王受不了了：算了，这些房间都不能刻名了，只能挂书包 。 

##### sychronized进阶原理

synchronized默认是使用轻量级锁，轻量级锁发生抢占时会升级为重锁。然后阻塞队列可以通过自旋优化来尽可能减少阻塞

###### 1.轻量级锁

轻量级锁的使用场景:如果一个对象虽然有多线程访问，但多线程访问的时间是错开的(也就是没有竞争)，那么可以使用轻量级锁来优化。

轻量级锁对使用者是透明的，即语法仍然是synchronized

假设有两个方法同步块，利用同一个对象加锁

```java
static final object obj = new object();
public static void method1() {
    synchronized( obj ) {
        //同步块A
        method2();
    }
}

public static void method2() {
    synchronized( obj ) {
        //同步块B
    }
}
```

- 创建锁记录(Lock Record)对象,每个线程都的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的`Mark Word`![轻量级锁](img\轻量级锁.jpg)

- 让锁记录中Object reference指向锁对象，并尝试用cas替换Object的Mark Word,将Mark Word的值存入锁记录![轻量级锁2](img\轻量级锁2.jpg)

- 如果`cas`(compare and swap)替换成功，对象头中存储了`锁记录地址和状态00`的，表示由该线程给对象加锁，这时图示如下![轻量级锁3](img\轻量级锁3.jpg)

- 如果cas失败,有两种情况
  - 如果是其它线程已经持有了该Object的轻量级锁，这时表明有竞争，进入锁膨胀过程
  - 如果是自己执行了synchronized 锁重入（自己又给自己对象加锁了，见下），那么再添加一条Lock Record作为重入的计数
    - 见轻量级锁示例代码：t0执行`syn method1(obj)`，获得锁之后继续调用`syn method2(obj)`（多出来一个栈帧，见下图），两个加锁的`obj`是同一个对象，因此`CAS`失败
    
    - 在图中的体现：对象头`lock record 地址 00`在调用`method1(obj)`改变了，指向的是第一个栈帧的锁记录，因此第二个栈帧会CAS失败
    
    - `Lock Record`的null记录锁重入的计数，如上为1，再调用一次++、
    
      ![轻量级锁4](img\轻量级锁4.jpg)
    
    - 当退出synchronized代码块(解锁时)如果有取值为null的锁记录，表示有重入，这时重置锁记录，表示重入计数减1
    
      ![轻量级锁5](img\轻量级锁5.jpg)
    
    - 当退出synchronized代码块(解锁时) 锁记录的值不为null,这时使用cas将Mark Word的值恢复给对象头
    
      - 成功，则解锁成功
      - 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程

###### 2.锁膨胀

如果在尝试加轻量级锁的过程中，CAS操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁(有竞争)，这时需要进行锁膨胀，将轻量级锁变为重量级锁。

```java
static Object obj = new Object();
public static void method1() {
    synchronized( obj ) {
        //同步块
    }
}
```

- 当Thread-1进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

![锁膨胀1](img\锁膨胀1.jpg)

- 这时Thread-1加轻量级锁失败，进入锁膨胀流程
  - 即为Object 对象申请Monitor锁,让Object指向重量级锁地址
  - 然后自己进入Monitor的EntryList BLOCKED

![锁膨胀2](img\锁膨胀2.jpg)

- 当Thread-0退出同步块解锁时，使用cas将Mark Word的值恢复给对象头，失败（此时锁膨胀了）。这时会进入重量级解锁流程，即按照Monitor地址找到Monitor对象，设置Owner为null，唤醒EntryList中BLOCKED线程

###### 3.自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞

**自旋重试成功的情况**

> 自旋需要cpu资源，所以适合多核cpu

| 线程1 (cpu1上)          | 对象Mark               | 线程2 (cpu2上)          |
| ----------------------- | ---------------------- | ----------------------- |
| -                       | 10 (重量锁)            | -                       |
| 访问同步块，获取monitor | 10 (重量锁) 重量锁指针 | -                       |
| 成功（加锁）            | 10 (重量锁) 重量锁指针 | -                       |
| 执行同步块              | 10 (重量锁) 重量锁指针 | -                       |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 访问同步块，获取monitor |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 自旋重试                |
| 执行完毕                | 10 (重量锁) 重量锁指针 | 自旋重试                |
| 成功（解锁）            | 无锁                   | 自旋重试                |
| -                       | 10 (重量锁) 重量锁指针 | 成功（加锁）            |
| -                       | 10 (重量锁) 重量锁指针 | 执行同步块              |
| -                       | ...                    | ...                     |

**自旋重试失败的情况**

| 线程1 (cpu1上)          | 对象Mark               | 线程2 (cpu2上)          |
| ----------------------- | ---------------------- | ----------------------- |
| -                       | 10 (重量锁)            | -                       |
| 访问同步块，获取monitor | 10 (重量锁) 重量锁指针 | -                       |
| 成功（加锁）            | 10 (重量锁) 重量锁指针 | -                       |
| 执行同步块              | 10 (重量锁) 重量锁指针 | -                       |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 访问同步块，获取monitor |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 自旋重试                |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 自旋重试                |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 自旋重试                |
| 执行同步块              | 10 (重量锁) 重量锁指针 | 阻塞                    |
| -                       | ...                    | ...                     |

- 在Java 6之后自旋锁是自适应的，比如对象刚刚的- -次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次;反之，就少自旋甚至不自旋，总之，比较智能。
- 自旋会占用CPU时间，单核CPU自旋就是浪费，多核CPU自旋才能发挥优势。
- Java 7之后不能控制是否开启自旋功能

###### 4.偏向锁

轻量级锁在没有竞争时(就自己这个线程)，每次重入仍然需要执行 CAS操作。

Java 6中引入了偏向锁来做进一步优化：只有第一次使用CAS将线程ID设置到对象的Mark Word头，之后发现这个线程ID是自己的就表示没有竞争，不用重新CAS。以后只要不发生竞争，这个对象就归该线程所有

例：

```java
static final object obj = new object();
public static void m1() {
    synchronized( obj ) {
        //同步块A
        m2();
    }
}
public static void m2() {
    synchronized( obj ) {
        //同步块B
        m3();
    }
}
public static void m3() {
    synchronized( obj ) {
        //同步块C
    }
}
```

<img src="img\偏向锁1.jpg" alt="偏向锁1" style="zoom:67%;" />

<img src="img\偏向锁2.jpg" alt="偏向锁2" style="zoom:67%;" />

**偏向状态**

https://www.bilibili.com/video/BV16J411h7Rd?t=14&p=83

回忆一下对象头格式

```ruby
|--------------------------------------------------------------------|--------------------|
| Mark Word (64 bits)                                                |       State        |
|--------------------------------------------------------------------|--------------------|
| unused:25 | hashcode:31 | unused:1 | age:4 | biased_lock:0 | 01    |       Normal       |
|--------------------------------------------------------------------|--------------------|
| thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | 01        |       Biased       |
|--------------------------------------------------------------------|--------------------|
| ptr_to_lock_record:62 | 00 | Lightweight                           |       Locked       |
|--------------------------------------------------------------------|--------------------|
| ptr_to_heavyweight_monitor:62 | 10 |                               | Heavyweight Locked |
|--------------------------------------------------------------------|--------------------|
|                                                            | 11    |    Marked for GC   |
|--------------------------------------------------------------------|--------------------|
```

一个对象创建时:

- 如果开启了偏向锁(默认开启)，那么对象创建后，`markword` 值为`0x05`即最后3位为101,这时它的
  `thread、epoch、 age` 都为0
- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加VM参数`-XX:BiasedLockingStartupDelay=0` 来禁用延迟
- 如果没有开启偏向锁，那么对象创建后，`markword` 值为`0x01`即最后3位为001,这时它的`hashcode、age`
  都为0，第一次用到`hashcode`时才会赋值

1) 测试延迟特性
2) 测试偏向锁

- 利用jol第三方工具来查看对象头信息(注意这里我扩展了jol让它输出更为简洁)

```java
//添如虚拟机参数-XX:BiasedLockingStartupDelay=0
    public static void main(String[] args) throws IOException {
        Dog d = new Dog();
        ClassLayout classLayout = ClassLayout.lparseInstance(d);
        new Thread(() -> {
            log.debug("synchronized前");
            System.out.println(classLayout.toPrintableSimple(true));
            synchronized (d) {
                log.debug("synchronized中");
                System.out.println(classlayout.toPrintableSimple(true));
            }
            log.debug(" synchraoized后");
            System.out.println(classLayout.toPrintablesimple(true));
        }, "t1").start();
    }
```

输出

```
11:08:58.117 c. TestBiased [t1] - synchronized 前
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000101
11:08:58.121 C. TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101
11:08:58.121 C. TestBiased [t1] - synchronized 后
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101

```

> 处于偏向锁的对象解锁后,线程 id仍存储于对象头中

3) 测试禁用

在上面测试代码运行时在添加VM参数 `-XX: -UseBiasedLocking` 禁用偏向锁

输出

```
11:13:10.018 c.TestBiased [t1] - synchronized 前
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
11:13:10.021 C. TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00100000 00010100 11110011 10001000
11:13:10.021 C. TestBiased [t1] - synchronized 后
```
4)测试hashcode

```java
public static void main(String[] args) throws IOException {
    Dog d = new Dog();
    d.hashcode();//调用对象hashcode，使得偏向锁禁用
    ClassLayout classLayout = ClassLayout.lparseInstance(d);
    new Thread(() -> {
        log.debug("synchronized前");
        System.out.println(classLayout.toPrintableSimple(true));
        synchronized (d) {
            log.debug("synchronized中");
            System.out.println(classlayout.toPrintableSimple(true));
        }
        log.debug(" synchraoized后");
        System.out.println(classLayout.toPrintablesimple(true));
    }, "t1").start();
}
```

> 观察如上的MarkWord格式，Normal下的hashcode占31位，Biased下的thread:54位，装不下hashcode。所以，可偏向对象调了hashcode()后撤销偏向状态
>
> 轻量级锁：hashcode会存到线程栈帧的锁记录(lock Record)中
>
> 重量级锁：hashcode会存到monitor对象中

**撤销偏锁1-调用对象hashCode**

调用了对象的hashCode，但偏向锁的对象MarkWord中存储的是线程id,如果调用hashCode会导致偏向锁被撤销

- 轻量级锁会在锁记录中记录hashCode
- 重量级锁会在Monitor中记录hashCode

在调用hashCode后使用偏向锁，记得去掉`-XX: -UseBiasedLocking`

输出：

```
11:22:10.386 c.TestBiased [main] - 调用hashCode: 1778535015
11:22:10.391 c.TestBiased [t1] - synchronized 前
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001
11:22:10.393 C. TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00100000 11000011 11110011 01101000
11:22:10.393 c.TestBiased [t1] - synchronized 后
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001
```

**撤销偏锁2-其它线程使用对象**

当有其它线程使用偏向锁对象时，会将偏向锁升级为轻量级锁

```java
public class Demo10 {
    private static void test2() throws InterruptedException {
        Dog d = new Dog();
        Thread t1 = new Thread(() -> {
            synchronized (d) {
                log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            synchronized (TestBiased.class) {
                TestBiased.class.notify();
            }
            //如果不用wait/notify 使用join必须打开下面的注释
            //因为: t1线程不能结束，否则底层线程可能被jvm重用作为t2线程，底层线程id是一样的
            /*try {
            System. in.read(); .
            } catch (IOException e) {
            e. printStackTrace();
            }*/
        }, "t1");
        t1.start();

        Thread t2 = new Thread(() -> {
            synchronized (TestBiased.class) {
                try {
                    TestBiased.class.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
            synchronized (d) {
                log.debug(Classlayout.parseInstance(d).toPrintableSimple(true));
            }
            log.debug(Class Layout.parseInstance(d).toPrintableSimple(true));
        }, "t2");
        t2.start();
    }
}
```

输出：

```
[t1] - 0000000 00000000 00000000 0000000 00011111 .01000001 00010000  00000101
[t2] - 00000000 00000000 0000000 0000000 00011111 01000001  00010000  00000101
[t2] - 00000000 0000000 00000000 0000000 00011111 10110101  11110000  01000000 //撤销偏向锁，改为轻量级锁，保留线程id
[t2] - 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 //恢复正常
```

**撤销偏锁3-调用wait/notify**

wait/notify只有重锁才有，任何线程对象调用其时，会升级位重锁

**批量重偏向**

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程T1的对象仍有机会重新偏向T2,重偏向会重置对象的Thread ID

当撤销偏向锁阈值超过20次后, jvm会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至加锁线程

状态转化：

偏向锁t1 -> t2加入竞争 ->有了竞争，不符合偏向t1了 -> 对于t2，先撤销t1偏锁，再升级轻锁，然后解锁变为不可偏向状态 ->t2连续上步，达到阈值20后 -> jvm默认只有t2了，偏向t2

代码演示

```java
public class Demo10 {
    public static void test() {
        Vector<Dog> list = new Vector<>();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 30; i++) {
                Dog d = new Dog();
                list.add(d);
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
                }
            }
            synchronized (list) {
                list.notify();//唤醒list
            }
        }, "t1");
        t1.start();
        Thread t2 = new Thread(() -> {
            synchronized (list) {
                try {
                    list.wait();//阻塞list，释放锁
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("===========> ");
            for (int i = 0; i < 30; i++) {
                Dog d = list.get(i);
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintablesimple(true));
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintablesimple(true));
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
                }
            }
        }, "t2");
        t2.start();
    }
}
```
输出
```java
xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx 线程id    线程id   线程id    加锁状态
[t1] - 0
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t1] - 1
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t1] - 2
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t1] - 3
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t1] - 4
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t1] - 5
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101

...t1 从1到29都是加的线程id（00011111 11101011）偏向锁，状态看最后101

[t2] - ============>
[t2] - 0
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101 //原始t1的偏向锁状态
[t2] - 0
00000000 00000000 00000000 00000000 00100000 01111010 11110110 01110000 //撤销偏向锁，升级轻量级锁
[t2] - 0
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001//解锁后，变为不可偏向状态
[t2] - 1
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t2] - 1
00000000 00000000 00000000 00000000 00100000 01111010 11110110 01110000
[t2] - 1
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
...
//我们发现，到了第20个的时候（从0算第1个），又变成了偏向锁状态，但是偏向的id变成了t2了
//之后所有的对象都是直接偏向的状态，而不是先撤销t1偏锁，再升级轻锁 => 批量重偏向
[t2] - 19
00000000 00000000 00000000 00000000 00011111 11101011 01000000 00000101
[t2] - 19
00000000 00000000 00000000 00000000 00011111 11101011 01010001 00000101
[t2] - 19
00000000 00000000 00000000 00000000 00011111 11101011 01010001 00000101
```

**批量撤销**

当撤销偏向锁阈值超过40次后，jvm 会这样觉得，自己确实偏向错了, 根本就不该偏向。于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的

```java
public class Demo11 {
    static Thread t1, t2, t3;

    public static void test() {
        Vector<Dog> list = new Vector<>();
        int loopNumber = 39;
        t1 = new Thread(() -> {
            for (int i = 0; i < loopNumber; i++) {
                Dog d = new Dog();
                list.add(d);
                //39个对象加上偏向锁，偏向t1线程
                synchronized (d) {
                    log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
                }
                //39个对象加完锁唤醒t2(park，unpark方式)
                LockSupport.unpark(t2);
            }
        }, "t1");
        t1.start();
        t2 = new Thread(() -> {
            LockSupport.park();//先阻塞自己
            log.debug("============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);//拿出list对象
                Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
                //对象加上偏向锁，偏向t2线程
                //前19个对象是撤销t1偏向锁，之后对象是批量重偏向
                synchronized (d) {
                    Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintablesimple(true));
                }
                Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            //此时已经重偏向了20次
            LockSupport.unpark(t3);//唤醒t3

        }, "t2");
        t2.start();
        t3 = new Thread(() -> {
            LockSupport.park();//先阻塞自己
            log.debug("============> ");
            for (int i = 0; i < loopNumber; i++) {
                Dog d = list.get(i);//拿出list对象
                Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
                //对象加上偏向锁，偏向t3线程
                //前19个对象是撤销t2偏向锁，注意：之后对象也是撤销t2偏锁，没那么多机会重偏向锁了
                synchronized (d) {
                    Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintablesimple(true));
                }
                Log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            //最后撤销偏向锁达到39次
        }, "t3");
        t3.start();

        t3.join();
        /*
         当撤销偏向锁阈值超过40次后，jvm会这样觉得，自己确实偏向错了，根本就不该偏向。
         于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的，所以new Dog()是不可偏向的
        */
        Log.debug(ClassLayout.parseInstance(new Dog()).toPrintableSimple(true));
    }
}
```

> 批量重偏向与撤销是针对类的优化与对象无关

###### 5.锁消除

案例：

```java
@Fork(1)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class Demo12 {
    static int x = 0;

    @Benchmark
    public void a() throws Exception {
        x++;
    }
    @Benchmark
    //JIT 即时编译器
    //对热点代码(如循环)，超过一定阈值，对代码进行优化
    public void b () throws Exception {
        object o = new object();//o对象是b()的局部变量，没有竞争
        //加锁和不加锁都一样，所以实际执行时JIT就把锁消除了
        synchronized (o) {
            x++;
        }
    }
}
```

`java -jar benchmarks.jar`（打包执行）

```java
Benchmark          Mode      Samples   Score     Score error  Units
c.i. MyBenchmark.  a avgt       5      1.542     0.056        ns/op
c.i. MyBenchmark.  b avgt       5      1.518     0.091        ns/op
//score值，方法执行时间，越小性能越高，可以看出差不多的
```

`java -XX:-EliminateLocks -jar benchmarks.jar` 关闭锁消除

```
Benchmark          Mode      Samples   Score     Score error  Units
c.i. MyBenchmark.  a avgt       5      1.542     0.018        ns/op
c.i. MyBenchmark.  b avgt       5      16.976    1.572        ns/op
```

### 4.7 `wait`& `notify`

#### 4.7.1 小故事 - 为什么需要 wait

- 由于条件不满足，小南不能继续进行计算
- 但小南如果一直占用着锁，其它人就得一直阻塞，效率太低
- ![wait](img\wait.jpg)
- 于是老王单开了一间休息室（调用 wait 方法），让小南到休息室（WaitSet）等着去了，但这时锁释放开，其它人可以由老王随机安排进屋，直到小M将烟送来，大叫一声 [ 你的烟到了 ] （调用 notify 方法）  
- ![wait2](img\wait2.jpg)
- 小南于是可以离开休息室，重新进入竞争锁的队列  
- ![wait3](img\wait3.jpg)

#### 4.7.2  `wait`& `notify`原理

![wait4](img\wait4.jpg)

- Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态
- BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片
- BLOCKED 线程会在 Owner 线程释放锁时唤醒
- WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入EntryList 重新竞争  

**join 原理（不详细）**

是调用者轮询检查线程 alive 状态  

```java
t1.join();
```

等价于下面的代码

```java
synchronized (t1) {
    // 调用者线程进入 t1 的 waitSet 等待, 直到 t1 运行结束
    while (t1.isAlive()) {
        t1.wait(0);
    }
}
```

> join 体现的是【保护性暂停】模式，请参考之  

#### 4.7.3 `wait`& `notify` API

| API             | 功能描述                                        |
| --------------- | ----------------------------------------------- |
| obj.wait()      | 让进入 object 监视器的线程到 waitSet 等待       |
| obj.notify()    | 在 object 上正在 waitSet 等待的线程中挑一个唤醒 |
| obj.notifyAll() | 让 object 上正在 waitSet 等待的线程全部唤醒     |

> 它们都是线程之间进行协作的手段，**都属于 Object 对象的方法**
>
> **必须获得此对象的锁**，才能调用这几个方法  

```java
final static Object obj = new Object();
public static void main(String[] args) {
    new Thread(() -> {
        synchronized (obj) {
            log.debug("执行....");
            try {
                obj.wait(); // 让线程在obj上一直等待下去
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("其它代码....");
        }
    }).start();
    new Thread(() -> {
        synchronized (obj) {
            log.debug("执行....");
            try {
                obj.wait(); // 让线程在obj上一直等待下去
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("其它代码....");
        }
    }).start();
    // 主线程两秒后执行
    sleep(2);
    log.debug("唤醒 obj 上其它线程");
    synchronized (obj) {
        obj.notify(); // 随机唤醒obj上一个线程
        // obj.notifyAll(); // 唤醒obj上所有等待线程
    }
}
```

notify 的一种结果

```
20:00:53.096 [Thread-0] c.TestWaitNotify - 执行....
20:00:53.099 [Thread-1] c.TestWaitNotify - 执行....
20:00:55.096 [main] c.TestWaitNotify - 唤醒 obj 上其它线程
20:00:55.096 [Thread-0] c.TestWaitNotify - 其它代码....
```

notifyAll 的结果  

```
19:58:15.457 [Thread-0] c.TestWaitNotify - 执行....
19:58:15.460 [Thread-1] c.TestWaitNotify - 执行....
19:58:17.456 [main] c.TestWaitNotify - 唤醒 obj 上其它线程
19:58:17.456 [Thread-1] c.TestWaitNotify - 其它代码....
19:58:17.456 [Thread-0] c.TestWaitNotify - 其它代码....
```

- `wait()` 方法会释放对象的锁，进入 WaitSet 等待区，从而让其他线程就机会获取对象的锁。无限制等待，直到notify为止

- `wait(long n)` 有时限的等待, 到 n 毫秒后结束等待，或是被 notify

#### 4.7.4 wait notify 的正确姿势

`sleep(long n)` **和** `wait(long n)` **的区别

- sleep 是 Thread 方法，而 wait 是 Object 的方法 
- sleep 不需要强制和 synchronized 配合使用，但 wait 需要和 synchronized 一起用
- sleep 在睡眠的同时，不会释放对象锁的，但 wait 在等待的时候会释放对象锁
- 它们状态 TIMED_WAITING    

##### step1

```java
static final Object room = new Object();
static boolean hasCigarette = false;
static boolean hasTakeout = false;  
```

思考下面的解决方案好不好，为什么？  

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            sleep(2);
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        }
    }
}, "小南").start();
for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        synchronized (room) {
            log.debug("可以开始干活了");
        }
    }, "其它人").start();
}
sleep(1);
new Thread(() -> {
    // 这里能不能加 synchronized (room)？
    hasCigarette = true;
    log.debug("烟到了噢！");
}, "送烟的").start();
```

输出

```
20:49:49.883 [小南] c.TestCorrectPosture - 有烟没？[false]
20:49:49.887 [小南] c.TestCorrectPosture - 没烟，先歇会！
20:49:50.882 [送烟的] c.TestCorrectPosture - 烟到了噢！
20:49:51.887 [小南] c.TestCorrectPosture - 有烟没？[true]
20:49:51.887 [小南] c.TestCorrectPosture - 可以开始干活了
20:49:51.887 [其它人] c.TestCorrectPosture - 可以开始干活了
20:49:51.887 [其它人] c.TestCorrectPosture - 可以开始干活了
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了
```

- 其它干活的线程，都要一直阻塞，效率太低
- 小南线程必须睡足 2s 后才能醒来，就算烟提前送到，也无法立刻醒来
- 加了 synchronized (room) 后，就好比小南在里面反锁了门睡觉，烟根本没法送进门，main 没加synchronized 就好像 main 线程是翻窗户进来的
- 解决方法，使用 wait - notify 机制 

##### step2

思考下面的实现行吗，为什么？  

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            try {
                room.wait(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        }
    }
}, "小南").start();

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        synchronized (room) {
            log.debug("可以开始干活了");
        }
    }, "其它人").start();
}

sleep(1);

new Thread(() -> {
    synchronized (room) {
        hasCigarette = true;
        log.debug("烟到了噢！");
        room.notify();
    }
}, "送烟的").start();
```

输出

```
20:51:42.489 [小南] c.TestCorrectPosture - 有烟没？[false]
20:51:42.493 [小南] c.TestCorrectPosture - 没烟，先歇会！
20:51:42.493 [其它人] c.TestCorrectPosture - 可以开始干活了
20:51:42.493 [其它人] c.TestCorrectPosture - 可以开始干活了
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了
20:51:43.490 [送烟的] c.TestCorrectPosture - 烟到了噢！
20:51:43.490 [小南] c.TestCorrectPosture - 有烟没？[true]
20:51:43.490 [小南] c.TestCorrectPosture - 可以开始干活了
```

- 解决了其它干活的线程阻塞的问题
- 但如果有其它线程也在等待条件呢  

##### step3

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            try {
                room.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        } else {
            log.debug("没干成活...");
        }
    }
}, "小南").start();
new Thread(() -> {
    synchronized (room) {
        Thread thread = Thread.currentThread();
        log.debug("外卖送到没？[{}]", hasTakeout);
        if (!hasTakeout) {
            log.debug("没外卖，先歇会！");
            try {
                room.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("外卖送到没？[{}]", hasTakeout);
        if (hasTakeout) {
            log.debug("可以开始干活了");
        } else {
            log.debug("没干成活...");
        }
    }
}, "小女").start();
sleep(1);
new Thread(() -> {
    synchronized (room) {
        hasTakeout = true;
        log.debug("外卖到了噢！");
        room.notify();
        //room.notifyAll();
    }
}, "送外卖的").start()
```

输出

```
20:53:12.173 [小南] c.TestCorrectPosture - 有烟没？[false]
20:53:12.176 [小南] c.TestCorrectPosture - 没烟，先歇会！
20:53:12.176 [小女] c.TestCorrectPosture - 外卖送到没？[false]
20:53:12.176 [小女] c.TestCorrectPosture - 没外卖，先歇会！
20:53:13.174 [送外卖的] c.TestCorrectPosture - 外卖到了噢！
20:53:13.174 [小南] c.TestCorrectPosture - 有烟没？[false]
20:53:13.174 [小南] c.TestCorrectPosture - 没干成活...
```

- notify 只能随机唤醒一个 WaitSet 中的线程，这时如果有其它线程也在等待，那么就可能唤醒不了正确的线程，称之为【虚假唤醒】
- 解决方法，改为 notifyAll  

##### step4

解开上步notifyAll

##### step5**

将 if 改为 while  

```java
if (!hasCigarette) {
    log.debug("没烟，先歇会！");
    try {
        room.wait();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

改动后  

```java
//if只能判断一次，判断完后结束程序，所以换成while
while (!hasCigarette) {
    log.debug("没烟，先歇会！");
    try {
        room.wait();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

输出  

```
20:58:34.322 [小南] c.TestCorrectPosture - 有烟没？[false]
20:58:34.326 [小南] c.TestCorrectPosture - 没烟，先歇会！
20:58:34.326 [小女] c.TestCorrectPosture - 外卖送到没？[false]
20:58:34.326 [小女] c.TestCorrectPosture - 没外卖，先歇会！
20:58:35.323 [送外卖的] c.TestCorrectPosture - 外卖到了噢！
20:58:35.324 [小女] c.TestCorrectPosture - 外卖送到没？[true]
20:58:35.324 [小女] c.TestCorrectPosture - 可以开始干活了
20:58:35.324 [小南] c.TestCorrectPosture - 没烟，先歇会！
```

wait & notify 模板代码

```java
synchronized(lock) {
    while(条件不成立) {
        lock.wait();
    }
    // 干活
}
//另一个线程
synchronized(lock) {
    lock.notifyAll();
}
```

# 97-113略

### 4.8 `Park` & `Unpark`  

### 4.9 重新理解线程状态转换  

### 4.10 多把锁

#### 4.10.1 多把不相干的锁  

一间大屋子有两个功能:睡觉、学习，互不相干。

现在小南要学习，小女要睡觉，但如果只用一间屋子(一个对象锁)的话，那么并发度很低

解决方法是准备多个房间(多个对象锁)

例子(粗粒度锁)：

> 粗粒度锁：负责不同性质的线程，用的是同一性质的锁对象。期间可能会并发性能差

```java
class BigRoom {
    public void sleep() {
        synchronized (this) {
            log.debug("sleeping 2 小时");
            Sleeper.sleep(2);
        }
    }
    public void study() {
        synchronized (this) {
            log.debug("study 1 小时");
            Sleeper.sleep(1);
        }
    }
}
```

执行  

```java
BigRoom bigRoom = new BigRoom();
new Thread(() -> {
    bigRoom.compute();
},"小南").start();
new Thread(() -> {
    bigRoom.sleep();
},"小女").start();
```

某次结果 

```
12:13:54.471 [小南] c.BigRoom - study 1 小时
12:13:55.476 [小女] c.BigRoom - sleeping 2 小时
```

改进(细粒度锁)：

> 细粒度锁：不同性质的线程，分配多个相应性质的锁对象与之对应。期间可能会造成死锁问题

```java
class BigRoom {
    private final Object studyRoom = new Object();
    private final Object bedRoom = new Object();
    public void sleep() {
        synchronized (bedRoom) {
            log.debug("sleeping 2 小时");
            Sleeper.sleep(2);
        }
    }
    public void study() {
        synchronized (studyRoom) {
            log.debug("study 1 小时");
            Sleeper.sleep(1);
        }
    }
}
```

某次执行结果

```
12:15:35.069 [小南] c.BigRoom - study 1 小时
12:15:35.069 [小女] c.BigRoom - sleeping 2 小时
```

将锁的粒度细分

- 好处，是可以增强并发度
- 坏处，如果一个线程需要同时获得多把锁，就容易发生死锁

### 4.11 活跃性

**活跃性**

线程内的代码是有限的，由于某种原因，线程内的代码一直执行不完。

活跃性分为以下几种情况

#### 4.11.1 死锁

有这样的情况：一个线程需要同时获取多把锁，这时就容易发生死锁

`t1 线程` 获得 `A对象`锁，接下来想获取`B对象`的锁`t2线程`获得`B对象`锁，接下来想获取`A对象`的锁 

例：    

```java
Object A = new Object();
Object B = new Object();
Thread t1 = new Thread(() -> {
    synchronized (A) {
        log.debug("lock A");
        sleep(1);
        synchronized (B) {
            log.debug("lock B");
            log.debug("操作...");
        }
    }
}, "t1");
Thread t2 = new Thread(() -> {
    synchronized (B) {
        log.debug("lock B");
        sleep(0.5);
        synchronized (A) {
            log.debug("lock A");
            log.debug("操作...");
        }
    }
}, "t2");
t1.start();
t2.start();
```
结果  

```
12:22:06.962 [t2] c.TestDeadLock - lock B
12:22:06.962 [t1] c.TestDeadLock - lock A
```

#### 4.11.2 定位死锁(实操)  

- 检测死锁可以使用 `jconsole`工具，或者使用 `jps` 定位进程 id，再用 `jstack` 定位死锁：  

```java
cmd > jps
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
12320 Jps
22816 KotlinCompileDaemon
33200 TestDeadLock // JVM 进程
11508 Main
28468 Launcher
```
```java
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.191-b12 mixed mode):

"DestroyJavaVM" #14 prio=5 os_prio=0 tid=0x0000000002d8e800 nid=0x1a14 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"t2" #13 prio=5 os_prio=0 tid=0x000000001fb08800 nid=0x3ef8 waiting for monitor entry [0x000000002049f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at demo.part1.Demo13.lambda$main$1(Demo13.java:38)
        - waiting to lock <0x000000076bb6ae88> (a java.lang.Object)
        - locked <0x000000076bb6ae98> (a java.lang.Object)
        at demo.part1.Demo13$$Lambda$2/2093631819.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

"t1" #12 prio=5 os_prio=0 tid=0x000000001fb05800 nid=0x3f54 waiting for monitor entry [0x000000002039e000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at demo.part1.Demo13.lambda$main$0(Demo13.java:24)
        - waiting to lock <0x000000076bb6ae98> (a java.lang.Object)
        - locked <0x000000076bb6ae88> (a java.lang.Object)
        at demo.part1.Demo13$$Lambda$1/558638686.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

//略去部分输出

//发现死锁
Found one Java-level deadlock:
=============================
"t2":
  waiting to lock monitor 0x000000001cd50a98 (object 0x000000076bb6ae88, a java.lang.Object),
  which is held by "t1"
"t1":
  waiting to lock monitor 0x000000001cd533d8 (object 0x000000076bb6ae98, a java.lang.Object),
  which is held by "t2"

Java stack information for the threads listed above:
===================================================
"t2":
        at demo.part1.Demo13.lambda$main$1(Demo13.java:38)
        - waiting to lock <0x000000076bb6ae88> (a java.lang.Object)
        - locked <0x000000076bb6ae98> (a java.lang.Object)
        at demo.part1.Demo13$$Lambda$2/2093631819.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)
"t1":
        at demo.part1.Demo13.lambda$main$0(Demo13.java:24)
        - waiting to lock <0x000000076bb6ae98> (a java.lang.Object)
        - locked <0x000000076bb6ae88> (a java.lang.Object)
        at demo.part1.Demo13$$Lambda$1/558638686.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.

```

#### 4.11.3 哲学家就餐

<img src="img\哲学家就餐.jpg" alt="哲学家就餐" style="zoom:67%;" />

有五位哲学家，围坐在圆桌旁

- 他们只做两件事，思考和吃饭，思考一会吃口饭，吃完饭后接着思考
- 吃饭时要用两根筷子吃，桌上共有 5 根筷子，每位哲学家左右手边各有一根筷子
- 如果筷子被身边的人拿着，自己就得等待  

筷子类

```java
class Chopstick {
    String name;
    public Chopstick(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```

哲学家类

```java
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;
    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }
    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }
    @Override
    public void run() {
        while (true) {
            // 获得左手筷子
            synchronized (left) {
                // 获得右手筷子
                synchronized (right) {
                    // 吃饭
                    eat();
                }
                // 放下右手筷子
            }
            // 放下左手筷子
        }
    }
}
```

  就餐

```java
Chopstick c1 = new Chopstick("1");
Chopstick c2 = new Chopstick("2");
Chopstick c3 = new Chopstick("3");
Chopstick c4 = new Chopstick("4");
Chopstick c5 = new Chopstick("5");
new Philosopher("苏格拉底", c1, c2).start();
new Philosopher("柏拉图", c2, c3).start();
new Philosopher("亚里士多德", c3, c4).start();
new Philosopher("赫拉克利特", c4, c5).start();
new Philosopher("阿基米德", c5, c1).start();
```

执行不多会，就执行不下去了  

```
12:33:15.575 [苏格拉底] c.Philosopher - eating...
12:33:15.575 [亚里士多德] c.Philosopher - eating...
12:33:16.580 [阿基米德] c.Philosopher - eating...
12:33:17.580 [阿基米德] c.Philosopher - eating...
// 卡在这里, 不向下运行
```

使用 jconsole 检测死锁，发现  

```properties
-------------------------------------------------------------------------
名称: 阿基米德
状态: cn.itcast.Chopstick@1540e19d (筷子1) 上的BLOCKED, 拥有者: 苏格拉底
总阻止数: 2, 总等待数: 1
堆栈跟踪:
cn.itcast.Philosopher.run(TestDinner.java:48)
- 已锁定 cn.itcast.Chopstick@6d6f6e28 (筷子5)
-------------------------------------------------------------------------
名称: 苏格拉底
状态: cn.itcast.Chopstick@677327b6 (筷子2) 上的BLOCKED, 拥有者: 柏拉图
总阻止数: 2, 总等待数: 1
堆栈跟踪:
cn.itcast.Philosopher.run(TestDinner.java:48)
- 已锁定 cn.itcast.Chopstick@1540e19d (筷子1)
-------------------------------------------------------------------------
名称: 柏拉图
状态: cn.itcast.Chopstick@14ae5a5 (筷子3) 上的BLOCKED, 拥有者: 亚里士多德
总阻止数: 2, 总等待数: 0
堆栈跟踪:
cn.itcast.Philosopher.run(TestDinner.java:48)
- 已锁定 cn.itcast.Chopstick@677327b6 (筷子2)
-------------------------------------------------------------------------
名称: 亚里士多德
状态: cn.itcast.Chopstick@7f31245a (筷子4) 上的BLOCKED, 拥有者: 赫拉克利特
总阻止数: 1, 总等待数: 1
堆栈跟踪:
cn.itcast.Philosopher.run(TestDinner.java:48)
- 已锁定 cn.itcast.Chopstick@14ae5a5 (筷子3)
-------------------------------------------------------------------------
名称: 赫拉克利特
状态: cn.itcast.Chopstick@6d6f6e28 (筷子5) 上的BLOCKED, 拥有者: 阿基米德
总阻止数: 2, 总等待数: 0
堆栈跟踪:
cn.itcast.Philosopher.run(TestDinner.java:48)
- 已锁定 cn.itcast.Chopstick@7f31245a (筷子4)
```

这种线程没有按预期结束，执行不下去的情况，归类为【活跃性】问题，除了死锁以外，还有活锁和饥饿者两种情
况  

#### 4.11.4 活锁

活锁出现在两个线程互相改变对方的结束条件，最后谁也无法结束，例如

```java
public class TestLiveLock {
    static volatile int count = 10;
    static final Object lock = new Object();
    public static void main(String[] args) {
        new Thread(() -> {
            // 期望减到 0 退出循环
            while (count > 0) {
                sleep(0.2);
                count--;
                log.debug("count: {}", count);
            }
        }, "t1").start();
        new Thread(() -> {
            // 期望超过 20 退出循环
            while (count < 20) {
                sleep(0.2);
                count++;
                log.debug("count: {}", count);
            }
        }, "t2").start();
    }
}
```

  解决方法：执行时间有一定交错、睡眠的时间是一个随机数

#### 4.11.5 饥饿

很多教程中把饥饿定义为，一个线程由于优先级太低，始终得不到 CPU 调度执行，也不能够结束，饥饿的情况不易演示，讲读写锁时会涉及饥饿问题

下面我讲一下我遇到的一个线程饥饿的例子，先来看看使用顺序加锁的方式解决之前的死锁问题
 ![时序图-饥饿](img\时序图-饥饿.jpg)

 顺序加锁的解决方案  
 ![时序图-饥饿2](img\时序图-饥饿2.jpg)

### 4.12 ReentrantLock（可重入锁）  

- 相对于 synchronized 它具备如下特点
  - 可中断
  - 可以设置超时时间
    - synchronized：线程A拥有锁，那么线程B只能进入等待队列，而ReentrantLock可以设置等待超时时间，规定时间获取不到锁，那么就放弃执行其他逻辑
  - 可以设置为公平锁
    - 公平锁是一种防止线程饥饿的情况，比如先进先出的规则
  - 支持多个条件变量
    - 比如waitset，可以设置多个waitset条件变量

- 与 synchronized 一样，都支持可重入

- synchronized是在关键字级别保护临界区，reentrantLock是在对象级别保护临界区

基本语法

```java
// 获取锁
reentrantLock.lock();
try {
    // 临界区
} finally {
    // 释放锁
    reentrantLock.unlock();
}
```

#### 4.12.1 可重入

可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁，如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住

> 通俗的说就是同个线程先后多次获得同一把锁  

```java
static ReentrantLock lock = new ReentrantLock();
public static void main(String[] args) {
    // method1()调method2()， method2()调method3()， method()内均有锁，当调用到method2、3()时，能够加锁成功，就称为可重入成功
    method1();
}
public static void method1() {
    //之前syn的方式，我们把synchronized(锁对象)中的锁对象当成锁，锁对象其实就是一个普通对象，锁实际是这个普通obj关联的monitor
    lock.lock();//锁对象：线程执行到这一步代码时，若可以获得锁对象，则成为锁的主人，否则进入等待队列
    try {
        log.debug("execute method1");
        method2();
    } finally {
        lock.unlock();
    }
}
public static void method2() {
    lock.lock();
    try {
        log.debug("execute method2");
        method3();
    } finally {
        lock.unlock();
    }
}
public static void method3() {
    lock.lock();
    try {
        log.debug("execute method3");
    } finally {
        lock.unlock();
    }
}
```

输出

```
17:59:11.862 [main] c.TestReentrant - execute method1
17:59:11.865 [main] c.TestReentrant - execute method2
17:59:11.865 [main] c.TestReentrant - execute method3
```

#### 4.12.2 可打断

示例

```java
ReentrantLock lock = new ReentrantLock();
public static void main(String args[]){
    Thread t1 = new Thread(() -> {
        log.debug("启动...");
        try {
            // lock.lock()方法是不可打断的
            // 如果没有竞争那么此方法就会获取 lock 对象锁
            // 如果有竞争就进入阻塞队列，不同于 lock 的阻塞队列，可以由其他线程用interrupt()打断，在阻塞队列唤醒线程，终止等待
            lock.lockInterruptibly();//可打断方法
        } catch (InterruptedException e) {
            e.printStackTrace();
            log.debug("等锁的过程中被打断");
            return;
        }
        try {
            log.debug("获得了锁");
        } finally {
            lock.unlock();
        }
    }, "t1");
    lock.lock();
    log.debug("获得了锁");
    t1.start();
    try {
        sleep(1);
        t1.interrupt();
        log.debug("执行打断");
    } finally {
        lock.unlock();
    }
}
```

输出

```
18:02:40.520 [main] c.TestInterrupt - 获得了锁
18:02:40.524 [t1] c.TestInterrupt - 启动...
18:02:41.530 [main] c.TestInterrupt - 执行打断
java.lang.InterruptedException
	at
java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at
java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at cn.itcast.n4.reentrant.TestInterrupt.lambda$main$0(TestInterrupt.java:17)
	at java.lang.Thread.run(Thread.java:748)
18:02:41.532 [t1] c.TestInterrupt - 等锁的过程中被打断
```

> 注意如果是不可中断模式，那么即使使用了 interrupt 也不会让等待中断  

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    lock.lock();
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");
lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(1);
    t1.interrupt();
    log.debug("执行打断");
    sleep(1);
} finally {
    log.debug("释放了锁");
    lock.unlock();
}
```

输出

```
18:06:56.261 [main] c.TestInterrupt - 获得了锁
18:06:56.265 [t1] c.TestInterrupt - 启动...
18:06:57.266 [main] c.TestInterrupt - 执行打断 // 这时 t1 并没有被真正打断, 而是仍继续等待锁
18:06:58.267 [main] c.TestInterrupt - 释放了锁
18:06:58.267 [t1] c.TestInterrupt - 获得了锁
```

#### 4.12.3 锁超时

立刻失败  

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    //lock.tryLock()尝试获得锁
    if (!lock.tryLock()) {
        log.debug("获取立刻失败，返回");
        return;
    }
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");
lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```
18:15:02.918 [main] c.TestTimeout - 获得了锁
18:15:02.921 [t1] c.TestTimeout - 启动...
18:15:02.921 [t1] c.TestTimeout - 获取立刻失败，返回
```

超时失败

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    try {
        if (!lock.tryLock(1, TimeUnit.SECONDS)) {
            log.debug("获取等待 1s 后失败，返回");
            return;
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");
lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```
18:19:40.537 [main] c.TestTimeout - 获得了锁
18:19:40.544 [t1] c.TestTimeout - 启动...
18:19:41.547 [t1] c.TestTimeout - 获取等待 1s 后失败，返回
```

##### 使用tryLock解决哲学家就餐问题  

```java
class Chopstick extends ReentrantLock {
    String name;
    public Chopstick(String name) {
        this.name = name;
    }
    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```

```java
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;
    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }
    @Override
    public void run() {
        while (true) {
            // 尝试获得左手筷子
            if (left.tryLock()) {
                try {
                    // 尝试获得右手筷子
                    if (right.tryLock()) {
                        try {
                            eat();
                        } finally {
                            right.unlock();
                        }
                    }
                } finally {
                    left.unlock();
                }
            }
        }
    }
    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }
}
```

#### 4.12.4 公平锁

`ReentrantLock` 默认是不公平的  

```java
ReentrantLock lock = new ReentrantLock(false);
lock.lock();
for (int i = 0; i < 500; i++) {
    new Thread(() -> {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " running...");
        } finally {
            lock.unlock();
        }
    }, "t" + i).start();
}
// 1s 之后去争抢锁
Thread.sleep(1000);
new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " start...");
    lock.lock();
    try {
        System.out.println(Thread.currentThread().getName() + " running...");
    } finally {
        lock.unlock();
    }
}, "强行插入").start();
lock.unlock();
```

强行插入，有机会在中间输出

> **注意**：该实验不一定总能复现

```
t39 running...
t40 running...
t41 running...
t42 running...
t43 running...
强行插入 start...
强行插入 running...
t44 running...
t45 running...
t46 running...
t47 running...
t49 running
```

改为公平锁后  

`ReentrantLock lock = new ReentrantLock(true); //设为公平锁 `

强行插入，总是在最后输出

```
t465 running...
t464 running...
t477 running...
t442 running...
t468 running...
t493 running...
t482 running...
t485 running...
t481 running...
强行插入 running...
```

> 公平锁一般没有必要，会**降低并发度**，后面分析原理时会讲解  

# 126-132略，看完97-113，wait notify再看

#### 4.12.5 条件变量  

synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待

ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比
- synchronized 是那些不满足条件的线程都在一间休息室等消息
- 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤
  醒  

使用流程：

- await 前需要获得锁
- await 执行后，会释放锁，进入 conditionObject 等待
- await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁
- 竞争 lock 锁成功后，从 await 后继续执行  

#### 4.12.6 同步模式之顺序控制  

### 本章小结

本章我们需要重点掌握的是

- 分析多线程访问共享资源时，哪些代码片段属于临界区
- 使用 synchronized 互斥解决临界区的线程安全问题
  - 掌握 synchronized 锁对象语法
  - 掌握 synchronzied 加载成员方法和静态方法语法
  - 掌握 wait/notify 同步方法
- 使用 lock 互斥解决临界区的线程安全问题
  - 掌握 lock 的使用细节：可打断、锁超时、公平锁、条件变量
- 学会分析变量的线程安全性、掌握常见线程安全类的使用
- 了解线程活跃性问题：死锁、活锁、饥饿
- 应用方面
  - 互斥：使用 synchronized 或 Lock 达到共享资源互斥效果
  - 同步：使用 wait/notify 或 Lock 的条件变量来达到线程间通信效果
- 原理方面
  - monitor、synchronized 、wait/notify 原理
  - synchronized 进阶原理
  - park & unpark 原理
- 模式方面
  - 同步模式之保护性暂停
  - 异步模式之生产者消费者
  - 同步模式之顺序控制  

