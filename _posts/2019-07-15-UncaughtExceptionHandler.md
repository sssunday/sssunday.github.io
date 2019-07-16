---
layout: post
title: UncaughtExceptionHandler - 处理非正常的线程中止
date: 2019-07-15
Author: sssunday
categories: 
tags: [java, 并发]
comments: true
---
#### 背景介绍
引用博客: <a href="https://blog.csdn.net/u013256816/article/details/50417822" target="_blank">JAVA多线程之UncaughtExceptionHandler——处理非正常的线程中止</a> 
<br>
当单线程的程序发生一个未捕获的异常时我们可以采用try....catch进行异常的捕获，但是在多线程环境中，线程抛出的异常是不能用try....catch捕获的，这样就有可能导致一些问题的出现，比如异常的时候无法回收一些系统资源，或者没有关闭当前的连接等等。

#### 1.异常处理器
+ 自定义一个异常处理器<strong>ThreadUncaughtExceptionHandler</strong>
    ```java
    public class ThreadUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {

        String handlerName;//处理器名称，方便跟踪是哪个处理器处理的异常
        public ThreadUncaughtExceptionHandler(String handlerName){
            this.handlerName = handlerName;
        }

        @Override
        public void uncaughtException(Thread t, Throwable e) {
            System.out.println(Thread.currentThread().getName() + " => " + handlerName + " get uncaught exception:" + e.getMessage());
        }
    }
    ```

#### 2.线程
+ 代码
    测试方法:

    ```java
    public static void main(String[] args) {
        //设置所有线程默认的异常处理器（全局异常处理器）
        Thread.setDefaultUncaughtExceptionHandler(new ThreadUncaughtExceptionHandler("handlerGlobal"));
        try{
            Thread t0 = getExceptionThread();
            //单独某一个线程的异常处理器,会覆盖掉全局的处理器
            t0.setUncaughtExceptionHandler(new ThreadUncaughtExceptionHandler("handlerCurrent"));
            t0.start();

            Thread t1 = getExceptionThread();
            t1.start();
            System.out.println(1/0);//此处异常会被捕获
        } catch (Exception e){
            System.out.println("main get exception:" + e.getMessage());
        }
        System.out.println(1/0);//此处异常会被全局异常处理器处理
    }

    static Thread getExceptionThread(){
        return new Thread(() -> {
            System.out.println(1/0);
        });

    }
    ```
+ 结果：
    ```java
    main get exception:/ by zero
    Thread-0 => handlerCurrent get uncaught exception:/ by zero
    Thread-1 => handlerGlobal get uncaught exception:/ by zero
    main => handlerGlobal get uncaught exception:/ by zero
    ```
+ 分析：
    + 1.main get exception:/ by zero 被捕获的异常，走到了catch块里面
    + 2.Thread-0 => handlerCurrent get uncaught exception:/ by zero 线程t0未被捕获的异常，由指定的handlerCurrent处理
    + 3.Thread-1 => handlerGlobal get uncaught exception:/ by zero 线程t1未被捕获的异常，由全局的handlerGlobal处理
    + 4.main => handlerGlobal get uncaught exception:/ by zero main线程未被捕获的异常，由全局的handlerGlobal处理
+ 结论：
    + 1.线程中的异常，只能在该线程内部捕获，调用线程（caller）无法捕获
    + 2.全局异常处理设置，会为所有线程设置异常处理器，包括main线程
    + 3.线程上的全局异常处理器会被该线程指定的异常处理器覆盖
    + 4.UncaughtExceptionHandler处理的是未被捕获的异常

#### 3.线程池
##### execute
+ 代码
    测试方法:

    ```java
    public static void main(String[] args) {
        //Thread.setDefaultUncaughtExceptionHandler(new ThreadUncaughtExceptionHandler("handlerGlobal"));
        //全局捕获器可生效
        //打开后执行结果为pool-1-thread-1 => handlerGlobal get uncaught exception:/ by zero
        ExecutorService exec = Executors.newCachedThreadPool();
        Thread t0 = getExceptionPoolThread();
        t0.setUncaughtExceptionHandler(new ThreadUncaughtExceptionHandler("handlerCurrent"));
        exec.execute(t0);
    }

    static Thread getExceptionPoolThread(){
        return new Thread(() -> {
            //Thread.currentThread().setUncaughtExceptionHandler(new ThreadUncaughtExceptionHandler("handlerCurrent"));
            //封装在Runnable和Callable中的指定捕获器可生效
            //打开后结果为：pool-1-thread-1 => handlerCurrent get uncaught exception:/ by zero
            System.out.println(1/0);
        });
    }
    ```
+ 结果：
    ```java
    Exception in thread "pool-1-thread-1" java.lang.ArithmeticException: / by zero
	at com.sssunday.blog.blog201907.testMain.lambda$getExceptionPoolThread$0(testMain.java:16)
	at java.lang.Thread.run(Thread.java:748)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
    ```
+ 分析：
    + 直接抛出未捕获异常，未被ThreadUncaughtExceptionHandler处理
    + 设置全局异常捕获处理器，可成功处理
+ 结论：
    + 1.execute前设置指定异常处理器不会生效，需要将捕获器封装在在Runnable或Callable中
    + 2.全局异常捕获器可成功处理

##### submit
+ 代码
    测试方法1: 不对异常做任何处理

    ```java
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        Thread t0 = getExceptionPoolThread();
        exec.submit(t0);
    }

    static Thread getExceptionPoolThread(){
        return new Thread(() -> {
            System.out.println("thread start");
            System.out.println(1/0);
            System.out.println("thread end");
        });
    }
    ```
    测试方法2: 使用Future的get方法重新抛出异常

    ```java
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        Thread t0 = getExceptionPoolThread();
        Future<?> future = exec.submit(t0);
        exec.shutdown();
        try {
            future.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }

    static Thread getExceptionPoolThread(){
        return new Thread(() -> {
            System.out.println("thread start");
            System.out.println(1/0);
            System.out.println("thread end");
        });
    }
    ```
+ 结果：
    ```java
    thread start
    java.util.concurrent.ExecutionException: java.lang.ArithmeticException: / by zero
        at java.util.concurrent.FutureTask.report(FutureTask.java:122)
        at java.util.concurrent.FutureTask.get(FutureTask.java:192)
        at com.sssunday.blog.blog201907.testMain.main(testMain.java:15)
    Caused by: java.lang.ArithmeticException: / by zero
        at com.sssunday.blog.blog201907.testMain.lambda$getExceptionPoolThread$0(testMain.java:24)
        at java.lang.Thread.run(Thread.java:748)
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
    ```
+ 分析：
    + 不做任何处理的时候，异常被吃掉了
    + Future的get方法又将异常重新抛出
+ 结论：
    + 1.通过submit提交的任务，无论是抛出的未检测异常还是已检查异常，都将被认为是任务返回状态的一部分。
    + 2.如果一个由submit提交的任务由于抛出了异常而结束，那么这个异常将被Future.get封装在ExecutionException中重新抛出。
