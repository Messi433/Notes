# JVM-垃圾回收

## 推荐博文

- https://blog.csdn.net/antony9118/article/details/51375662

## 三、垃圾回收

- [ ] 如何判断对象可以回收
- [ ] 垃圾回收算法
- [ ] 分代垃圾回收
- [ ] 垃圾回收器
- [ ] 垃圾回收调优

### 3.1 如何判断垃圾对象可以回收

#### 3.1.1 引用计数法

早期Python使用

两个对象相互引用(引用变量均为1)，但是实际并未引用，程序无法通过计数值为0来垃圾回收

![垃圾回收-引用计数法](img\垃圾回收-引用计数法.jpg)



#### 3.1.2 可达性分析算法

- Java 虚拟机中的垃圾回收器采用可达性分析来探索所有存活的对象
- 扫描堆中的对象，看是否能够沿着 GC Root对象 为起点的引用链找到该对象，找不到，表示可以回收；
- 根对象，肯定不能垃圾回收的；Java虚拟机中的垃圾回收器采用可达性分析来探索所有存活的对象；
- 扫描堆中的对象，看是否能够沿着GC Root对象为起点的引|用链找到该对象，找不到，表示可以回收哪些对象可以作为GC Root ?

##### Eclipse Memory Analyzer

查看gc roots

<img src="img\EclipseMemoryAnalyzer.jpg" alt="EclipseMemoryAnalyzer" style="zoom: 80%;" />

<img src="img\EclipseMemoryAnalyzer2.jpg" alt="EclipseMemoryAnalyzer2" style="zoom: 80%;" />

gc roots对象说明：

- SystemClass：系统类
- Native Stack：系统方法调用的Java对象
- Thread：活动线程调动的对象
- Busy Monitor：同步锁对象

​	

#### 3.1.3 四种引用

只有强引用是直接引用，其他引用都是间接引用，存在 ”中介对象“(软、弱、虚、终结器)

![垃圾回收-引用类型](img\垃圾回收-引用类型.jpg)

##### 1 强引用

- 只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收  
- 举例：new一个对象，等号赋值（建立引用），沿着GC  Root能找到此对象，则该对象不可被引用，与GC Root对象建立了强引用，如A1


##### 2 软引用

- 内存不足时，且对象没有强引用，可以被回收
- 触发：对象被垃圾回收时&内存不够时

##### 3 弱引用

- 对象没有强引用，即使内存充足也可被回收
- 触发：对象被垃圾回收时

##### 4 引用队列：

- 目的：软、弱、虚、终这些中介对象占用一定内存，当无对象进行引用他们时，会利用引用队列，一次遍历即可释放这些中介对象。

- 触发：当引用软、弱、虚、终这些中介对象的对象被回收时


##### 5 虚引用

- 软弱可以不借用引用队列，虚引用及终结器引用必须使用引用队列，虚引用主要配合 ByteBuffer 使用，被引用对象回收时，会将虚引用入队。
- 例如：创建ByteBuffer的时候会创建一个名为Cleaner的虚引用对象，当ByteBuffer没有被强引用所引用就会被jvm垃圾回收，虚引用Cleaner就会进入引用队列，会有专门的线程扫描引用队列，被发现后会调用直接内存地址的方法将直接内存释放掉，保证直接内存不会导致内存泄漏
- 由 Reference Handler 线程调用虚引用相关方法释放直接内存  

##### 6 终结器引用

- 如上A4对象，创建的时候会关联一个引用队列，且没有被强引用，当A4被垃圾回收的时候，会将终结器引用放入到一个引用队列（被引用对象暂时还没有被垃圾回收），有专门的线程（优先级较低，可能会造成对象迟迟不被回收）扫描引用队列并调用finallize()方法，第二次GC的时候才能回收掉被引用对象
- 无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象暂时没有被回收），再由 Finalizer 线程通过终结器引用找到被引用对象并调用它的 finalize方法，第二次 GC 时才能回收被引用对象。  

演示强引用、软引用



```java
/**
 * @Description 演示软引用：网上浏览图片，会存储到缓存中，此时若图片对象采用了强引用，会造成内存溢出
 * @Setting -Xmx20m -XX:+PrintGCDetails -verbose:gc
 */
public class Demo2 {
    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) throws IOException {
        //hard();//java.lang.OutOfMemoryError: Java heap space
        System.in.read();
        soft();
    }

    /***
     * @Description 演示强引用
     */
    public static void hard() {
        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(new byte[_4MB]);
        }
    }

    /***
     * @Description 演示弱引用
     */
    public static void soft() {
        List<SoftReference<byte[]>> list = new ArrayList<>();//list -> SoftReference -> byte[]，引用过程
        for (int i = 0; i < 5; i++) {
            //byte对象包装成软引用类型
            SoftReference<byte[]> reference = new SoftReference<>(new byte[_4MB]);
            System.out.println(reference.get());//打印软引用包装的对象
            list.add(reference);
            System.out.println(list.size());
        }
        System.out.println("循环结束：" + list.size());
        for (SoftReference<byte[]> reference : list) {
            System.out.println(reference.get());//打印软引用包装的对象，部分为null，被垃圾回收了
        }
    }
}
```

```properties
#打印日志如下
[B@1b6d3586
1
[B@4554617c
2
[B@74a14482
3
#第三次触发一次垃圾回收；PSYoungGen新生代，ParOldGen老年代
#Full GC,更全面的垃圾回收
#当一次Full GC回收后仍无足够内存，会释放引用软引用的对象。
[GC (Allocation Failure) [PSYoungGen: 2785K->480K(6144K)] 15073K->13360K(19968K), 0.0018255 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 480K->0K(6144K)] [ParOldGen: 12880K->13248K(13824K)] 13360K->13248K(19968K), [Metaspace: 3716K->3716K(1056768K)], 0.0112937 secs] [Times: user=0.11 sys=0.00, real=0.01 secs] 
[B@1540e19d
4
[Full GC (Ergonomics) [PSYoungGen: 4208K->0K(6144K)] [ParOldGen: 13248K->5000K(11776K)] 17457K->5000K(17920K), [Metaspace: 3716K->3716K(1056768K)], 0.0077984 secs] [Times: user=0.02 sys=0.01, real=0.01 secs] 
[B@677327b6
5

循环结束：5
null
null
null
[B@1540e19d
[B@677327b6
Heap
 PSYoungGen      total 6144K, used 4376K [0x00000000ff980000, 0x0000000100000000, 0x0000000100000000)
  eden space 5632K, 77% used [0x00000000ff980000,0x00000000ffdc6200,0x00000000fff00000)
  from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 11776K, used 5000K [0x00000000fec00000, 0x00000000ff780000, 0x00000000ff980000)
  object space 11776K, 42% used [0x00000000fec00000,0x00000000ff0e2040,0x00000000ff780000)
 Metaspace       used 3722K, capacity 4540K, committed 4864K, reserved 1056768K
  class space    used 409K, capacity 428K, committed 512K, reserved 1048576K
```

配合引用队列

```java
/**
 * @Description 演示软引用，配合软引用队列;
 * @PS 引用队列的作用：对象被垃圾回收时，没人引用软、弱、虚、终这些中介对象，则进入引用队列
 * @Setting -Xmx20m -XX:+PrintGCDetails -verbose:gc
 */
public class Demo3 {
    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) {
        soft();
    }

    /***
     * @Description 演示软引用
     */
    public static void soft() {
        List<SoftReference<byte[]>> list = new ArrayList<>();//list -> SoftReference -> byte[]，引用过程
        ReferenceQueue<byte[]> queue = new ReferenceQueue<>();//引用队列
        for (int i = 0; i < 5; i++) {
            //byte对象包装成软引用类型
            SoftReference<byte[]> reference = new SoftReference<>(new byte[_4MB], queue);
            System.out.println(reference.get());//打印软引用包装的对象
            list.add(reference);
            //期间会触发垃圾回收，使得软引用对象(中介)进入引用队列
            System.out.println(list.size());
        }
        Reference<? extends byte[]> poll = queue.poll();//获得最先进入队列的软引用对象
        //poll不为空，说明引用队列不为空
        while (poll != null) {
            list.remove(poll);//对象集合中，移除没被引用的软引用对象
            poll = queue.poll();
        }
        System.out.println("循环结束：" + list.size());
        //配合引用队列后，不会打印null值了，因为软引用对象被回收了，只有没被垃圾回收的对象及其引用的软引用对象
        for (SoftReference<byte[]> reference : list) {
            System.out.println(reference.get());//打印软引用包装的对象，部分为null，被垃圾回收了
        }
    }
}
```

弱引用演示

上面代码Soft改为Weak

### 3.2 垃圾回收算法

#### 3.2.1 标记清除算法

![垃圾回收-标记清除算法](img\垃圾回收-标记清除算法.jpg)

##### 步骤说明：

- 标记GC Root不引用的对象
- 清除对象

##### 优缺点：

- 速度快

- 清除垃圾后，不整理，易产生页内碎片(OS)

  > 假如需要创造一段连续空间（数组），则无法装入

#### 3.2.2 标记整理

区别于**标记清除**，在清除的过程中，会整理空间，使后面的引用对象向前移动，从而使得空间连续。

##### 优缺点：

- 速度慢，因为涉及对象移动
- 清除垃圾后，不整理，不易产生页内碎片(OS)

#### 3.2.3 复制

![垃圾回收-复制算法](img\垃圾回收-复制算法.jpg)

![垃圾回收-复制算法2](img\垃圾回收-复制算法2.jpg)

![垃圾回收-复制算法3](img\垃圾回收-复制算法3.jpg)

##### 步骤说明：

- 开辟FROM、TO两个内存空间
- 进行垃圾标记
- 将存活对象移动到TO
- FROM中执行垃圾回收
- 交换FROM、TO，恢复初态。

##### 优缺点：

- 不产生页内碎片(OS)
- 需开辟两个内存空间FROM、TO

### 3.3 分代垃圾回收

![分代回收-新老](img\分代回收-新老.jpg)

#### 3.3.1 分代的概念

新生代

- 用完就可以丢弃、生命周期较短、随时需要回收的对象
- 手纸，饭盒

老年代

- 长时间使用的对象，更有价值，长时间存活

- 老物件

#### 3.3.2 新生代-老年代的垃圾回收

![分代回收-新老步骤1](img\分代回收-新老步骤1.jpg)

![分代回收-新老步骤2](img\分代回收-新老步骤2.jpg)

![分代回收-新老步骤3](img\分代回收-新老步骤3.jpg)

![分代回收-新老步骤4](img\分代回收-新老步骤4.jpg)

![分代回收-新老步骤5](img\分代回收-新老步骤5.jpg)

##### 步骤说明：

- 对象首先分配在伊甸园区域

- 新生代空间不足时，触发 minor gc，伊甸园和 from 存活的对象使用 copy 复制到 to 中，存活的对象年龄加 1并且交换 from to

- minor gc 会引发 stop the world，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行，当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit）

- 当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gc，STW的时间更长  

  > 当要加入的内存对象太大，新生代内存不够，若老年代符合，则放入老年代。

##### 代码演示：

```java
/**
 * @Description 测试子线程内存溢出时是否会导致整个Java程序报错
 */
public class Demo4 {
    private static final int _8MB = 8 * 1024 *1024;
    //-Xms20M -Xmx20M -Xmn10M -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            ArrayList<byte[]> list= new ArrayList<>();
            list.add(new byte[_8MB]);
            list.add(new byte[_8MB]);
        }).start();
        //下述代码若取消睡眠后，则不影响整个程序
        System.out.println("sleep.........");
        Thread.sleep(100000000L);
    }
}
```

#### 3.3.3 垃圾回收参数说明

![垃圾回收-GC参数](img\垃圾回收-GC参数.jpg)



### 3.4 垃圾回收器

#### 3.4.1 串行

- 单线程
- 堆内存较小，适合个人电脑
- 虚拟机参数：`-XX:+UseSerialGC= Serial + SerialOld`
  - `Serial`：工作在新生代，复制算法
  - `SerialOld`：工作在老年代，标记+整理算法
- ![垃圾回收-串行](img\垃圾回收-串行.jpg)

  - 安全点：4个线程同时运行，期间内存不够触发垃圾回收，之后4个线程同时在**安全点**停下来。因为垃圾回收过程中，会改变内存地址，为了安全的使用这些地址，所以需要当前运行的线程在安全点停下来，此时完成垃圾回收工作就不会被干扰了。
  - 期间垃圾回收线程运行，其他线程阻塞。

#### 3.4.2 吞吐量优先

- 多线程

- 堆内存较大，多核cpu

- 让单位时间内，STW（响应时间）的时间最短 （少餐多食）
  
  - 0.2 0.2 = 0.4 
  
- 虚拟机参数：

  - `-XX:+UseParallelGC ~ -XX:+UseParallelOldGC` （1.8默认开启）
    - 新生代，复制算法；老年代，标记整理算法
    - `Parallel`：并行，多线程
  - `-XX:+UseAdaptiveSizePolicy`
    - 动态调整伊甸园与survivor(幸存)区，以及其他堆内存
  - `-XX:GCTimeRatio=ratio`
    - 设置吞吐量大小，它的值是一个 0-100 之间的整数。假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集（一般设置为19）
  - `-XX:MaxGCPauseMillis=ms`
    - 最大暂停毫秒数（默认200ms）

  > 上面两个参数所达到的效果是冲突的：`GCTimeRatio`增加，吞吐量增加，堆内存增加，暂停时间增加，`MaxGCPauseMillis`即最大暂停时间减小，吞吐量也就限制了。

  - ``-XX:ParallelGCThreads=n`
    - 垃圾回收线程数（默认是cpu核数）

- ![垃圾回收-吞吐量优先](img\垃圾回收-吞吐量优先.jpg)



#### 3.4.3 响应时间优先

- 多线程
- 堆内存较大,多核cpu
- 尽可能让单次STW的时间最短（少食多餐）
  - 0.1 0.1 0.1 0.1 0.1 = 0.5
  - 虚拟机参数：
    - `-XX:+UseConcMarkSweepGC ~ -XX:+UseParNewGC ~ SerialOld`
      - `concurrent`:并发，`MarkSweep`：标记清除
      - 用户进程与垃圾回收进程并发执行，抢占资源。
      - `-XX:+UseConcMarkSweepGC ~ -XX:+UseParNewGC`：成对执行，一个老年代，一个新生代。若并发失败，则退化成串行`SerialOld`回收方式。
    - `-XX:ParallelGCThreads=n ~ -XX:ConcGCThreads=threads`
      - 并行线程数 ~ 并发线程数
      - 并发线程数建议设置成并行线程数n的1/4
    - `-XX:CMSInitiatingOccupancyFraction=percent`
    - `-XX:+CMSScavengeBeforeRemark`
  - ![垃圾回收-响应时间优先](img\垃圾回收-响应时间优先.jpg)
    - 初始标记非常快，暂停时间短。
    - 重新标记，因为第二个并发运行阶段可能造成对象内存改变，所以重新标记。期间会线程阻塞
    - 只有初试标记和重新标记会造成其他线程阻塞，其余均并发执行。







