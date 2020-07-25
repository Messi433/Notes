# Java并发编程

## 1. 概览

### 1.1 这门课讲什么

这门课中的【并发】一词涵盖了在 Java 平台上的

- 进程
- 线程
- 并发
- 并行

以及 Java 并发工具、并发问题以及解决方案，同时我也会讲解一些其它领域的并发

### 1.2 为什么学这么课

- 我工作中用不到并发啊？  
  - 如果只是CURD，那用不到
  - 如果需要深层次理解，为之后阅读框架源码、拔高等打下基础，那用得到。

### 1.3 课程特色

- 本门课程以并发、并行为主线，穿插讲解
  - 应用 - 结合实际
  - 原理 - 了然于胸
  - 模式 - 正确姿势  

![学习路线1](img\学习路线1.jpg)
![学习路线2](img\学习路线2.jpg)
![学习路线3](img\学习路线3.jpg)

### 1.4 预备知识

- 希望你不是一个初学者
- 线程安全问题，需要你接触过 Java Web 开发、Jdbc 开发、Web 服务器、分布式框架时才会遇到
- 基于 JDK 8，最好对函数式编程、lambda 有一定了解
- 采用了 slf4j 打印日志，这是好的实践
- 采用了 lombok 简化 java bean 编写
- 给每个线程好名字，这也是一项好的实践  

### 1.5 相关配置文件

pom.xml

```xml
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.10</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
    </dependency>
</dependencies>
```

logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration xmlns="http://ch.qos.logback/xml/ns/logback"
               xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:schemaLocation="http://ch.qos.logback/xml/ns/logback logback.xsd">
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%date{HH:mm:ss} [%t] %logger - %m%n</pattern>
        </encoder>
    </appender>
    <logger name="c" level="debug" additivity="false">
        <appender-ref ref="STDOUT"/>
    </logger>
    <root level="ERROR">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

## 2.进程与线程

- [ ] 进程和线程的概念
- [ ] 并行和并发的概念
- [ ] 线程基本应用  

### 2.1 进程与线程

**进程**

- 程序由指令和数据组成，但这些指令要运行，数据要读写，就必须将指令加载至 CPU，数据加载至内存。在
  指令运行过程中还需要用到磁盘、网络等设备。进程就是用来加载指令、管理内存、管理 IO 的
- 当一个程序被运行，从磁盘加载这个程序的代码至内存，这时就开启了一个进程。
- 进程就可以视为程序的一个实例。大部分程序可以同时运行多个实例进程（例如记事本、画图、浏览器
  等），也有的程序只能启动一个实例进程（例如网易云音乐、360 安全卫士等）

**线程**

- 一个进程之内可以分为一到多个线程。
- 一个线程就是一个指令流，将指令流中的一条条指令以一定的顺序交给 CPU 执行
- Java 中，线程作为最小调度单位，进程作为资源分配的最小单位。 在 windows 中进程是不活动的，只是作
  为线程的容器  

**二者对比**

- 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集

- 进程拥有共享的资源，如内存空间等，供其内部的线程共享

- 进程间通信较为复杂

  - 同一台计算机的进程通信称为 IPC（Inter-process communication）
  - 不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP

- 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量

- 线程更轻量，线程上下文切换成本一般上要比进程上下文切换低

  > 上下文切换：指的是内核在CPU上对进程或者线程进行切换；内存总是有限的，当资源紧张时，优先切换给需要资源的进程或者线程

- 引入线程前，进程是资源调度和分配的基本单位，引入线程后，进程是操作系统资源分配的基本单位，而线程是任务调度和执行的基本单位

### 2.2 并行与并发  

单核 cpu 下，线程实际还是**串行执行** 。操作系统中有一个组件叫做任务调度器，将 cpu 的时间片（windows
下时间片最小约为 15 毫秒）分给不同的程序使用，只是由于 cpu 在线程间（时间片很短）的切换非常快，人类感
觉是 同时运行的 。总结为一句话就是**微观串行，宏观并行** ，

一般会将这种 线程轮流使用 CPU 的做法称为**并发**（**concurrent**）

1.单核cpu调度

| CPU  | 时间片 1 | 时间片 2 | 时间片 3 | 时间片 4 |
| ---- | -------- | -------- | -------- | -------- |
| core | 线程 1   | 线程 2   | 线程 3   | 线程 4   |

![cpu调度线程1](img\cpu调度线程1.jpg)

2.多核cpu调度

多核 cpu下，每个 核（core） 都可以调度运行线程，这时候线程可以是并行的  

| CPU    | 时间片 1 | 时间片 2 | 时间片 3 | 时间片 4 |
| ------ | -------- | -------- | -------- | -------- |
| core 1 | 线程 1   | 线程 1   | 线程 3   | 线程 3   |
| core 2 | 线程 2   | 线程 4   | 线程 2   | 线程 4   |

![cpu调度线程2](img\cpu调度线程2.jpg)

引用 Rob Pike 的一段描述：

- 并发（concurrent）是同一时间应对（dealing with）多件事情的能力

- 并行（parallel）是同一时间动手做（doing）多件事情的能力

例子

- 家庭主妇做饭、打扫卫生、给孩子喂奶，她一个人轮流交替做这多件事，这时就是并发
- 家庭主妇雇了个保姆，她们一起这些事，这时既有并发，也有并行（这时会产生竞争，例如锅只有一口，一个人用锅时，另一个人就得等待） 
- 雇了3个保姆，一个专做饭、一个专打扫卫生、一个专喂奶，互不干扰，这时是并行   

### 2.3  应用

#### 应用之异步调用（案例1）

以调用方角度来讲，如果：

- 需要等待结果返回，才能继续运行就是同步
- 不需要等待结果返回，就能继续运行就是异步

（1）设计

多线程可以让方法执行变为异步的（即不要巴巴干等着）比如说读取磁盘文件时，假设读取操作花费了 5 秒钟，如果没有线程调度机制，这 5 秒 cpu 什么都做不了，其它代码都得暂停...

（2）结论

- 比如在项目中，视频文件需要转换格式等操作比较费时，这时开一个新线程处理视频转换，避免阻塞主线程
- tomcat 的异步 servlet 也是类似的目的，让用户线程处理耗时较长的操作，避免阻塞 tomcat 的工作线程
- ui 程序中，开线程进行其他操作，避免阻塞 ui 线程  

#### 应用之提高效率（案例1）

充分利用多核 cpu 的优势，提高运行效率。想象下面的场景，执行 3 个计算，最后将计算结果汇总  

```
计算 1 花费 10 ms
计算 2 花费 11 ms
计算 3 花费 9 ms
汇总需要 1 ms
```

- 如果是串行执行，那么总共花费的时间是 `10 + 11 + 9 + 1 = 31ms`
- 但如果是四核 cpu，各个核心分别使用线程 1 执行计算 1，线程 2 执行计算 2，线程 3 执行计算 3，那么 3 个线程是并行的，花费时间只取决于最长的那个线程运行的时间，即 `11ms` 最后加上汇总时间只会花费 `12ms`  

> 需要在多核 cpu 才能提高效率，单核仍然时是轮流执行  

（1）设计

> 代码见【应用之效率-案例1】

（2）结论

- 单核 cpu 下，多线程不能实际提高程序运行效率，只是为了能够在不同的任务之间切换，不同线程轮流使用cpu ，不至于一个线程总占用 cpu，别的线程没法干活
- 多核 cpu 可以并行跑多个线程，但能否提高程序运行效率还是要分情况的
  - 有些任务，经过精心设计，将任务拆分，并行执行，当然可以提高程序的运行效率。但不是所有计算任务都能拆分（参考后文的【阿姆达尔定律】）
  - 也不是所有任务都需要拆分，任务的目的如果不同，谈拆分和效率没啥意义
- IO 操作不占用 cpu，只是我们一般拷贝文件使用的是【阻塞 IO】，这时相当于线程虽然不用 cpu，但需要一
  直等待 IO 结束，没能充分利用线程。所以才有后面的【非阻塞 IO】和【异步 IO】优化  

## 3.Java线程

- 创建和运行线程
- 查看线程
- 线程 API
- 线程状态  

### 3.1 创建和运行线程

#### 方法一，直接使用 Thread  

```java
// 创建线程对象
Thread t = new Thread() {
    public void run() {
        // 要执行的任务
    }
};
// 启动线程
t.start();
```

例如：  

```java
// 构造方法的参数是给线程指定名字，推荐
Thread t1 = new Thread("t1") {
    @Override
    // run 方法内实现了要执行的任务
    public void run() {
        log.debug("hello");
    }
};
t1.start();
```

输出 

`19:19:00 [t1] c.ThreadStarter - hello  ` 

#### 方法二，使用 Runnable 配合 Thread

把【线程】和【任务】（要执行的代码）分开

- Thread 代表线程
- Runnable 可运行的任务（线程要执行的代码）

```java
Runnable runnable = new Runnable() {
    public void run(){
        // 要执行的任务
    }
};
// 创建线程对象
Thread t = new Thread( runnable );
// 启动线程
t.start();
```

例如：  

```java
// 创建任务对象
Runnable task2 = new Runnable() {
    @Override
    public void run() {
        log.debug("hello");
    }
};
// 参数1 是任务对象; 参数2 是线程名字，推荐
Thread t2 = new Thread(task2, "t2");
t2.start();
```

输出：

`19:19:00 [t2] c.ThreadStarter - hello`

##### Java 8 以后可以使用 lambda 精简代码

```java
// 创建任务对象
Runnable task2 = () -> log.debug("hello");
// 参数1 是任务对象; 参数2 是线程名字，推荐
Thread t2 = new Thread(task2, "t2");
t2.start();
```

Runable接口源码

```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see java.lang.Thread#run()
     */
    public abstract void run();
}
```

> 只有一个抽象方法的接口，会有`@FunctionalInterface`注解，`lambda`表达式只能简化这种注解的接口

```java
@Slf4j(topic = "test1")
public class Demo2 {
    public static void main(String[] args) {
        /*Runnable r = new Runnable() {//idea alter+enter 提示
            @Override
            public void run() {
                System.out.println("run1");
            }
        };*/

        //lambda表达式简化
        Runnable r = () ->{
            //里面的抽象方法run也省略了(因为只有一个run)
            System.out.println("lambda run");
        };
        Thread t = new Thread(r,"t1");
        t.start();
    }
}
```

#### *原理之 Thread 与 Runnable 的关系

分析 Thread 的源码，理清它与 Runnable 的关系

- 方法1 是把线程和任务合并在了一起，方法2 是把线程和任务分开了
- 用 Runnable 更容易与线程池等高级 API 配合
- 用 Runnable 让任务类脱离了 Thread 继承体系，更灵活  

#### 方法三，FutureTask 配合 Thread

FutureTask 能够接收 Callable 类型的参数，用来处理有返回结果的情况

```java
public class Demo3 {
    public static void main(String[] args) {
        FutureTask<Integer> task = new FutureTask<>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println("thread running");
                Thread.sleep(2000);//休眠2s
                return 100;
            }
        });
        //FutureTask实现了runnable接口，自然可以作为参数传入
        Thread t1 = new Thread(task,"t1");
        t1.start();
        try {
            System.out.println(task.get());//获得task返回值，在休眠两秒后获得
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

```java
/*lambda简化*/
public static void main(String[] args) {
    // 创建任务对象
    FutureTask<Integer> task3 = new FutureTask<>(() -> {
        log.debug("hello");
        return 100;
    });
    // 参数1 是任务对象; 参数2 是线程名字，推荐
    new Thread(task3, "t3").start();
    // 主线程阻塞，同步等待 task 执行完毕的结果
    Integer result = task3.get();
    log.debug("结果是:{}", result);
}
```

输出  

`19:22:27 [t3] c.ThreadStarter - hello`
`19:22:27 [main] c.ThreadStarter - 结果是:100  `

### 3.2 多个线程同时运行  

主要是理解

- 交替执行
- 谁先谁后，不由我们控制  

