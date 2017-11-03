---
layout: post
title: "Callback Java"
description: "callback explained by java"
category: "java"
tags: [java,learning]
---

# 回调——Java实现

## 引子

在编写Android应用的时候，总会遇到这样一个情况，有一个比较大的资源，你需要把它下载下来。下载完后对它进行处理，然后再在界面上做出相应的显示，这里的资源可能是图片、大段的json/xml。

一般步骤是下载，处理，展示。其中下载与处理比较花费时间。
希望达到的效果是：用户进入有这样步骤的界面时，界面元素中能展示的尽量提前或者占位展示，而需要经过上述步骤才能展示的可以等到下载处理完成后再显示。这样做能不会影响用户的操作与体验。

Android与界面改变有关的操作需要在主线程中完成，费时的操作需要另开线程来完成，处理完后的结果由主线程来展示。这里就要用到回调以及异步了。  

## 什么是回调机制？

- 专业点说就是：A调用B中的某个方法b，然后B反过来调用A中的方法a，其中方法a就是回调方法。
- 通俗点说就是：A有一件事不方便自己完成，于是让B帮忙，B完成后将结果通知A。

这里A把事情交给B后有两种可能性：

1. A等到B通知结果才继续做下面的事
2. A把事情交给B后继续做其它的事，结果返回时再来继续完成

这就是同步和异步的区别了。  


## 分析抽象

从上面的描述中可以抽象出实现回调需要的基本元素：

1. 类A，需要做事的人
2. 类B，帮A做事的人
3. 接口Callback，B的反馈渠道

那么如何将两人联系起来呢？

1. 由于A需要B帮忙，那么A一定要知道B是谁，于是A中需要有B的引用；
2. B完事后需要通知A，这里需要A提供一个反馈渠道，及回调函数。考虑到低耦合性，这里将回调函数声明为一个接口。

简练的说就是：

1. class A实现Callback接口
2. class A有class B的引用b
3. class B有一个将Callback作为参数的方法，func(callback,...)
4. A的对象a调用B的方法func(callback,...)
5. b在func(callback,...)调用Callback接口中的方法

这里a在调用方法时，是在当前线程还是另开一个线程就是同步、异步的区别了。


## Talk is cheap.Show me the code.

**同步实例**：有一个“数盲”，它从某个地方获得了一个数字，想知道这个数字的奇偶性，但是由于……（人艰不拆），他只好依靠身边的一台机器来告诉它结果。

Callback.java

```java
public interface Callback {
    public void publishResult(String content);//向众人宣布结果
}
```

Worker.java

```java
public class Worker implements Callback{
    private Machine machine = new Machine();
    
    @Override
    public void publishResult(String content) {
        System.out.println("The result number is "+content+"!");
    }

    public int generateNum(){
        Random random = new Random();
        return random.nextInt(100);
    }

    //分析
    public void analysis(){
        int statistic = generateNum();
        System.out.println("the XXX is "+statistic+".");
        machine.doTask(this,statistic);
    }
    
}
```

Machine.java

```java
public class Machine {
    //真正在分析的是我
    public void doTask(Callback callback,int statistic){
        try {
            Thread.sleep(3000);
            if(statistic%2 == 0){
                callback.publishResult("even");
            }else{
                callback.publishResult("odd");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**异步实例**：本地向远端发送了一个信息交给它处理，它可以不断的发信息。

Callback.java

```java
public interface Callback {
    public void execute(Object... objects);
}
```
Local.java

```java
public class Local implements Callback, Runnable {

    private Remote remote = new Remote();
    private String message;

    public void setMessage(String message) {
        this.message = message;
    }

    public void sendMessage() {
        Thread thread = new Thread(this, message);
        thread.start();
        System.out.println("Message has been sent!");
    }

    @Override
    public void run() {
        remote.processMsg(message, this);
    }

    @Override
    public void execute(Object... objects) {
        System.out.println("Thread:" + Thread.currentThread().getName() + "~"
                + "Returning msg:" + objects[0]+"\n");
    }
}
```

Remote.java

```java
public class Remote {
    public void processMsg(String msg, Callback callback) {
        System.out.println("processing...\n");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("the message is " + msg);
        callback.execute(new String[] { "job has beens done" });
    }
}
```




