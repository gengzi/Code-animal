[TOC]

## 目标

* 了解**CountDownLatch**

  参考：[countDownLatch](https://www.jianshu.com/p/e233bb37d2e6)

  [CountDownLatch的介绍和使用](https://www.itzhai.com/the-introduction-and-use-of-a-countdownlatch.html)

## 概念

* CountDownLatch 允许一个或多个线程等待其他线程完毕后再执行

* 流程：

  通过一个计数器来设置需要完成的线程数，每个线程完成任务会让计数器减1，直到所有线程完成，计数器变0 。在让等待其他线程完成的线程再执行。

## 使用场景

* 通用的同步工具，将CountDownLatch初始化为1，可以看做一道门，其他线程都在门口等待，直到这道门开启，在执行其他线程。
* 可以将问题分为n个部分，用每个线程（Runnable）来执行每个部分，CountDownLatch监控所有线程完成，再执行某个流程。

## 方法详情

参考jdk1.8 中文api

参考：[countDownLatch](https://www.jianshu.com/p/e233bb37d2e6)

构造方法：

```java
//参数count为计数值
public CountDownLatch(int count) {  };
```

一般方法：

```java
// 调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public void await() throws InterruptedException { }; 
 
// 和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };
 
// 将count值减1
public void countDown() { };
 
// 返回当前计数
long getCount()
 
//返回一个标识此锁存器的字符串及其状态。
String toString()
 
```

##  示例

针对上面的两种使用场景

### 场景1

```java
import lombok.SneakyThrows;
import java.util.concurrent.CountDownLatch;
 
/**
* 其他线程等待门打开，再执行
*/
public class EmergencyDoor {
 
    @SneakyThrows
    public static void main(String[] args) {
        // 创建门
        CountDownLatch countDownLatch = new CountDownLatch(1);
        // 创建其他线程
        OtherRunnable otherRunnable = new OtherRunnable(countDownLatch);
        for (int i = 0; i < 100; i++) {
            new Thread(otherRunnable).start();
        }
        System.out.println("等待30秒");
        // 等待三十秒
        Thread.sleep(30000);
        // 打开门，现在其他线程就开始执行了
        countDownLatch.countDown();
    }
}
 
class OtherRunnable implements Runnable {
    private CountDownLatch downLatch;
    private volatile int count = 0;
 
    public OtherRunnable(CountDownLatch downLatch) {
        this.downLatch = downLatch;
    }
 
    @SneakyThrows
    @Override
    public void run() {
        downLatch.await();
        // 执行一些事情
        System.out.println("执行" + (count++));
    }
}
 
```

###  场景2

```java
 
import lombok.SneakyThrows;
import java.util.concurrent.CountDownLatch;
 
/**
* 主线程等待其他线程执行完毕，再执行
*/
public class Concurrentprocessing {
 
 
    @SneakyThrows
    public static void main(String[] args) {
        int count = 0;
        // 创建计数器
        CountDownLatch countDownLatch = new CountDownLatch(100);
        // 执行其他线程
        OtherRun otherRun = new OtherRun(countDownLatch, count);
        for (int i = 0; i < 100; i++) {
            new Thread(otherRun).start();
        }
        // 等待其他线程执行完毕
        countDownLatch.await();
        System.out.println("执行主线程");
    }
}
 
class OtherRun implements Runnable {
    private CountDownLatch downLatch;
    private volatile int count = 0;
 
    public OtherRun(CountDownLatch downLatch, int count) {
        this.downLatch = downLatch;
        this.count = count;
    }
 
    @SneakyThrows
    @Override
    public void run() {
        synchronized (downLatch) {
            // 执行一些事情
            count += 1;
            System.out.println("执行" + count);
            // 将计数器减1
            downLatch.countDown();
        }
    }
}
```

 

参考：

[性能优化-记ThreadPoolExecutor和CountDownLatch的一次实际优化经历](https://www.jianshu.com/p/77e978ac2432)

[使用线程池与CountDownLatch多线程提升系统性能](https://blog.csdn.net/exceptional_derek/article/details/52234640)

[接口RT优化场景，内部服务并行调用](https://www.jianshu.com/p/f17692e9114f)



 

 

 