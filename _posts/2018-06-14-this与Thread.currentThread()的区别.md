---
title:      this与Thread.currentThread()的区别
date:       2018-06-14
key: 20180614
author:     ChayCao
catalog: true
tags:  Java          
---


## 引言

首先来看下下面这段代码。这是一段简单的中断线程的示例代码。

```java
public class Test {

    static class Runner extends Thread {
        @Override
        public void run() {
            while (true) {
                if (Thread.currentThread().isInterrupted()) {
                    System.out.println("Interruted!");
                    break;
                }
            }
        }
    }
    
    public static void main(String[] args) {
        Runner runner = new Runner();
        runner.start();
        runner.interrupt();
    }

}
```

其中的 “**Thread.currentThread()**” 引发了我的思考：“既然该Runner的对象正在作为线程运行，那 this 和 Thread.currentThread() 不也就是同一个对象，为什么不直接用this呢？，相比之下this更加简短。”

在上面这段段代码中，将 Thread.currentThread() 替换成 this，是等同的，运行结果相同，能正常退出。

而下面代码，this 和 Thread.currentThread() 则运行结果则不同。this 会出现死循环，无法退出；Thread.currentThread() 能正常退出。

```java
public class Test {

    static class Runner extends Thread {
        @Override
        public void run() {
            while (true) {
                if (this.isInterrupted()) {
                    System.out.println("Interruted!");
                    break;
                }
            }
        }
    }

    public static void main(String[] args) {
        Runner runner = new Runner();
        Thread thread = new Thread(runner);
        thread.start();
        thread.interrupt();
    }

}
```

这说明 this 和 Thread.currentThread() 并不能完全等同。

其原因主要跟 Thread 类的内部实现有关。



## 正文 

在直接揭秘两者区别之前，先来了解下 Thread 类的内部实现。

Runner的中实现了构造函数，main方法中只创建了 runner 实例，并没有启动线程。

```java
public class Test {
    static class Runner extends Thread {
        public Runner() {
            System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
            System.out.println("Thread.currentThread().isAlive()=" + Thread.currentThread().isAlive());
            System.out.println("this.getName=" + this.getName());
            System.out.println("this.isAlive()=" + this.isAlive());
        }
    }

    public static void main(String[] args) {
        Runner runner = new Runner();
    }
}

/** output
Thread.currentThread().getName()=main
Thread.currentThread().isAlive()=true
this.getName=Thread-0
this.isAlive()=false
**/
```

根据前两行输出，可以看出，“当前的执行线程是 main 线程，并且处于运行状态”，容易理解。

再看后两行输出，this 指向的是新建的 runner 实例，该实例的 name 属性为 “Thread-0”，并且没有运行。没有运行很容易理解，因为没有执行 runner.start() 方法。关于 name 属性为何是 “Thread-0”，这是在创建 Thread 实例时，初始化的名字。

下面是 Thread类 的构造函数，其中 init() 方法第三个参数为线程的默认名字。生成名称的规则是：“Thread-”加上创建的线程的个数（第几个）。默认从0开始，由于main线程是默认就有的，所以并不计数 。

```java
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
```

所以 runner.getname() 得到的结果是 "Thread-0"。

---

接下来再看一段代码。新建 Runner 实例对象 runner。由于 Runner 是 Thread 的子类，并且 Thread 实现了 Runnable 接口，所以可以作为 Thread 类的构造函数参数传入。将 runner 作为构造函数参数传入，创建 Thread 实例对象 thread，启动 thread。

```java
public class Test {
    static class Runner extends Thread {
        @Override
        public void run() {
            System.out.println("Thread.currentThread().getName()=" + Thread.currentThread().getName());
            System.out.println("Thread.currentThread().isAlive()=" + Thread.currentThread().isAlive());
            System.out.println("this.getName=" + this.getName());
            System.out.println("this.isAlive()=" + this.isAlive());
        }
    }

    public static void main(String[] args) {
        Runner runner = new Runner();
        Thread thread = new Thread(runner);
        thread.start();
    }
}

/** output
Thread.currentThread().getName()=Thread-1
Thread.currentThread().isAlive()=true
this.getName=Thread-0
this.isAlive()=false
**/
```

首先看下前两行输出，可看出 Thread.currentThread() 指向 thread 对象，其name为“Thread-1”（第二个被创建），并且处于运行状态。

再来看看后两行输出，可看出 this 指向的是 runner 对象，其name为“Thread-0”，并且没有运行。

为什么 this 不是指向 thread 呢？？？ 不是 thread 在运行吗？？？

其原因在于 ```Thread thread = new Thread(runner)``` 会**将 runner 对象绑定到 thread 对象的一个 pravite 变量target 上，在 thread 被执行的时候即 thread.run() 被调用的时候，它会调用 target.run() 方法**，也就是说它是直接调用 runner 对象的run方法。

再确切的说，在run方法被执行的时候，this.getName() 实际上返回的是 target.getName()，而Thread.currentThread().getName() 实际上是 thread.getName()。


```java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```



## 总结

正在执行线程中的 this 对象并非就一定是该执行线程，因为执行线程的 run方法 可能是直接调用 其他Runnabel对象的 run() 方法。

个人看法，不要在线程中用 this 来表示当前运行线程，不可靠。



## 参考文献

1. [Java多线程之this与Thread.currentThread()的区别——java多线程编程核心技术](https://www.cnblogs.com/huangyichun/p/6071625.html)