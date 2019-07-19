---
layout: post
title: 简单的抽取ThreadPoolExcutor的逻辑
date: 2019-07-18
Author: sssunday
categories: 
tags: [java, 线程池]
comments: true
---
##### 代码
    
   ```java
    public class ThreadPoolExcutorDemo {
        int workerCount = 10;//线程数
        //线程集合
        private final HashSet<Worker> workers = new HashSet<Worker>();
        //任务队列
        private final BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<Runnable>(1024);
        void addWorker(Runnable task){
            if(workers.size() < 10){
                Worker worker = new Worker(task);
                workers.add(worker);
                worker.t.start();
            } else {
                workQueue.offer(task);
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