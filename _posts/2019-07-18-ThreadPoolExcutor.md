---
layout: post
title: 简单的抽取ThreadPoolExcutor的逻辑
date: 2019-07-18
Author: sssunday
categories: 
tags: [java, 线程池]
comments: true
---
ThreadPoolExcutor源码抽取其中最简单的逻辑
一个大小为10的线程池
线程池满了就往队列加任务

##### 代码
    
   ```java
    public class ThreadPoolExcutorDemo {
        int coreCount = 10;//核心线程数
        int maxCount = 15;//最大线程数
        //线程集合
        private final HashSet<Worker> workers = new HashSet<Worker>();
        //任务队列
        private final BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<Runnable>(1024);
        void addWorker(Runnable task){
            if(workers.size() < coreCount){
                Worker worker = new Worker(task);
                workers.add(worker);
                worker.t.start();
            } else if(!workQueue.offer(task) && workers.size() < maxCount){
                Worker worker = new Worker(task);
                workers.add(worker);
                worker.t.start();
            } else {
                System.out.println("线程池已满，任务队列已满");
            }
        }

        class Worker implements Runnable {

            final Thread t;
            Runnable task;

            public Worker(Runnable r) {
                task = r;
                t = new Thread(r);
            }

            @Override
            public void run() {
                Runnable task = this.task;
                while (task != null || (task = getTask()) != null) {
                    task.run();
                }
                if (getIsNeedTimeout()) {
                    this.t.interrupt();
                }
            }

            private boolean getIsNeedTimeout() {
                //是否超时，并且需要被干掉
                return true;
            }

            private Runnable getTask() {
                //从任务队列拿任务
                try {
                    return workQueue.take();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return null;
            }
        }
    }
  ```