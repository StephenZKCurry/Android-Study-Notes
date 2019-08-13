# Handler知识体系

> **前言**
>
> 在Android系统中，Handler扮演着非常重要的角色，通常所说的消息机制其实就是指的Handler。我们在开发中通常会使用Handler来实现子线程和主线程之间的通信，关于Handler的使用和实现机制可以说是老生常谈了，也有很多优秀的文章和资料可供参考。本文的目的就是站在巨人的肩膀上，力求对Handler的知识体系做一个尽量全面的总结梳理，也算是好好地复习一下。

[TOC]

## 1.Handler的简单使用

众所周知，Android主线程中不能进行耗时操作，诸如网络请求和文件操作这类耗时任务就需要创建子线程来执行，执行成功后往往需要更新UI，而子线程中又是不能进行UI操作的，这时候就轮到Handler出场了，它可以做到将线程切换到主线程进而来执行UI的相关操作。

Handler的使用相信我们都已经很熟悉了，简单来说就是发送和处理消息（好像是废话。。。）。首先需要在主线程中定义一个Handler对象，重写`handleMessage()`方法，在方法内进行消息的处理，可以根据消息的属性值来执行对应操作，比如请求数据成功进行展示。

```java
Handler mHandler = new Handler(){
  @Override
  public void handleMessage(Message msg) {
    // 接收并处理消息
  }
};
```

之后就可以发送消息来处理了，Handler发送消息的方式有两种：`sendMeaasge`和`post`，我们来分别看一下这两种方式的使用。

* **Handler.sendMessage(Message msg)**

```java
// 创建消息
Message message = Message.obtain();
message.what = 1;
message.obj = "StephenCurry";
// 发送消息
mHandler.sendMessage(message);
```

使用`Message.obtain()`方法创建一条消息，可以给Messgae设置一些属性值，常用的两个属性就是**what**和**obj**，what属性为int类型，一般作为一个标识；obj属性为Object类型，一般可以将网络请求返回的数据赋给obj属性。之后调用Handler的`sendMessage()`方法发送消息，参数传入创建好的消息对象。`sendMessage()`方法会返回一个布尔值，表示消息是否发送成功（准确地说是表示是否成功加入到了消息队列中，关于消息队列我后面会详细讲到），发送消息成功后，这条消息会在消息队列中排队，当消息被取出时，就会调用之前定义好的`handleMessage()`方法，这里可以根据message的属性值进行相关操作。由于Handler对象定义在主线程中，因此`handleMessage()`方法是在主线程中执行的，相当于实现了线程的切换，具体原理我后面会详细分析。需要注意的一点是，如果在消息处理之前已经退出了消息循环，这条消息就不会被处理。形象一点理解就是商场清仓甩卖，市民们（消息）排队去抢购，但是商品很快就卖完了（消息循环退出），没排到的市民（消息）就只能空手而归（消息不被处理）。

除了`sendMessage()`方法以外，Handler还提供了一系列**send**开头的方法来发送消息，这里简单列举一下。

>* **sendEmptyMessage(int what)**：发送空消息，该消息只有what属性
>* **sendEmptyMessageDelayed(int what, long delayMillis)**：指定延时多少毫秒后发送空消息
>* **sendEmptyMessageAtTime(int what, long uptimeMillis)**：指定在某个时间发送空消息
>* **sendMessageDelayed(Message msg, long delayMillis)**：指定延时多少毫秒后发送消息
>* **sendMessageAtTime(Message msg, long uptimeMillis)**：指定在某个时间发送消息
>* **sendMessageAtFrontOfQueue(Message msg)**：将消息添加到消息队列最前面

* **Handler.post(Runnable r)**

```java
mHandler.post(new Runnable() {
    @Override
    public void run() {
        // 执行相关操作，比如UI操作
    }
});
```

调用Handler的`post()`方法也可以发送并处理一条消息，参数传入一个Runnable对象，表示要执行的任务。与`sendMessage()`不同的是，我们将消息的处理逻辑写到了Runable的`run()`方法中，需要注意这里的Runnable和线程没有关系，`run()`方法中的逻辑并不是在子线程中运行的，因为Handler是在主线程中定义的，因此`run()`方法同样会在主线程中执行，同样实现了线程的切换，关于这点在后面我会详细分析，这里我们只需要清楚就好。

类似地，Handler也提供了一系列**post**开头的方法，这里先提一下，**post**系列方法实际上是把Runnable封装为了Message对象，最终还是调用的**send**系列方法。

> * **postAtTime(Runnable r,  long uptimeMillis)**：在指定时间发送消息
> * **postAtTime(Runnable r, Object token, long uptimeMillis)**：在指定时间发送消息，token参数会被赋给Message的obj属性
> * **postDelayed(Runnable r, long delayMillis)**：指定延时多少毫秒后发送消息
> * **postDelayed(Runnable r, Object token, long delayMillis)**：指定延时多少毫秒后发送消息，token参数会被赋给Message的obj属性
> * **postAtFrontOfQueue(Runnable r)**：将消息添加到消息队列最前面

Handler的简单实用就介绍得差不多了，当然我这里并没有展示太多的代码，我相信大家都很熟悉了。

## 2.相关类的介绍

在进行Handler工作原理分析之前，我先简单介绍一下相关的几个类，帮助我们首先建立一个初步的认识，在后面分析时就会清楚很多。

### 2.1.Handler

Handler是本文的主角，可译为消息处理者，负责将Message添加到消息队列中，当Looper从消息队列中取出Message时，就会分发给相应的Handler进行消息的处理，即调用Handler对象的`handleMessage()`方法。每个Handler创建时就相当于绑定了一个线程，`handleMessage()`方法会在创建Handler的线程中执行。

### 2.2.Looper

Looper表示消息循环，负责不断地从消息队列中取出Message，如果有消息则接着分发给Handler进行处理，没有就会一直阻塞在那里，直到有新消息添加到消息队列，每个线程只有一个Looper对象。

### 2.3.Message和MessageQueue

Message上面也简单介绍过了，是一个消息对象，包含了一些要发送给Handler的数据或对象，扮演着线程间的信使角色。每一个消息对象都对应着唯一一个Handler，可以认为是“绑定”了一个Handler对象，在处理消息时也会调用这个“绑定”的Handler对象的`handleMessage()`方法。MessageQueue译为消息队列，所有发送的Message都会被添加到MessageQueue中，它的内部实际上是一个链表结构，每一个Looper只有一个MessageQueue。

### 2.4.ThreadLocal

与上面的几个类不同，ThreadLocal其实不是Android特有的，它是Java中的一个类，这里我就详细介绍一下。ThreadLocal是一个线程内部的数据存储类，可以使用它在指定的线程中存储数据，而且只有该线程可以获取到存储的数据，其他线程是获取不到的，即使是使用同一个ThreadLocal对象。我们通过一个例子来看一下：

```java
private static ThreadLocal<String> threadLocal = new ThreadLocal<>();

public static void main(String[] args) {
    // 主线程中设置值
    threadLocal.set("MainThread的threadLocal");
    System.out.println(threadLocal.get());

    Thread thread1 = new Thread(new Runnable() {
        @Override
        public void run() {
            // 线程1中设置值
            threadLocal.set("Thread1的threadLocal");
            System.out.println(threadLocal.get());
        }
    });

    Thread thread2 = new Thread(new Runnable() {
        @Override
        public void run() {
            // 线程2中设置值
            threadLocal.set("Thread2的threadLocal");
            System.out.println(threadLocal.get());
        }
    });
    thread1.start();
    thread2.start();
}
```

可以看出，即使使用的是同一个ThreadLocal对象，在不同线程中通过`get()`方法获取到的值也是不一样的，这和我们在哪个线程中调用`set()`方法有关。



再说回Handler，ThreadLocal在消息机制中负责的是保存Looper对象。

上面几个类之间的关系：

* 一个线程只有一个Looper ，一个Looper只有一个`MessageQueue`
* 一个 Handler只能绑定一个Looper，但是一个Looper可以和多个Handler 相关联

## 3.Handler的工作原理

### 3.1.Handler与Looper的关联

Handler在创建时会先检查当前线程的Looper是否存在，如果不存在会抛出异常。

```java
public Handler(Callback callback, boolean async) {
        //检查当前的线程是否有Looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        //Looper持有一个MessageQueue
        mQueue = mLooper.mQueue;
}
```

如果Handler是在主线程中创建的就不需要我们手动创建Looper，因为主线程已经为我们创建好了一个Looper。如果要在子线程中创建Handler，需要记得先创建好Looper。

```java
class LooperThread extends Thread {
    public Handler mHandler;
    public void run() {
      	// 创建Looper
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // 接收并处理信息
            }
        };
      	// 开始不断尝试从MessageQueue中获取Message,并分发给对应的Handler
      	// loop()方法一个死循环
        Looper.loop();
    }
}
```

### 3.2.Message的存储与管理

Handler提供了一系列的方法让我们来发送消息，可以分为两大类：**send**系列**post**系列 。

不过不管我们调用什么方法，最终都会走到 `Message.enqueueMessage(Message,long)` 方法，这个方法的作用就是讲Message加入到MessageQueue（消息队列）中。

### 3.3.Message的分发和处理

`Looper.loop()` 负责从MessageQueue中循环获取Message并进行分发。

```java
// Looper
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    //...
  	// 死循环，唯一跳出循环的方式是MessageQueue的next()方法返回null
    for (;;) {
       // 不断从MessageQueue获取消息
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        //...
        try {
          	// 分发message
         	// msg.target就是发送消息的handler
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            //...
        }
        //...
		// 回收message
        msg.recycleUnchecked();
    }
}
```

最后调用了`msg.target.dispatchMessage(msg)` ，msg.target 就是发送该消息的 Handler，这样就回调到了 Handler 那边去了:

```java
// Handler
public void dispatchMessage(Message msg) {
  // msg.callback是Runnable，如果是post方法则会走这个if
  if (msg.callback != null) {
    handleCallback(msg);
  } else {
    // callback,构造Handler时可作为参数传入
    if (mCallback != null) {
      if (mCallback.handleMessage(msg)) {
        return;
      }
    }
    // 回调到Handler的handleMessage方法
    handleMessage(msg);
  }
}
```



## 4.Handler使用时需要注意的问题

### 4.1.Message的创建



### 4.2.内存泄漏



### 4.3.子线程中使用Toast



### 4.4.关于主线程Looper

#### 4.4.1.为什么主线程不需要创建Looper



#### 4.4.2.为什么Looper循环不会阻塞主线程



## 5.扩展知识

### 5.1.View.post()方法

大家可能都知道，在Activity的`onCreate()`方法中调用View的`getWidth()`或是`getHeight()`方法是获取不到View的宽高的，准确地说在`onStart()`和`onResume()`方法中也获取不到。原因的话我这里首先要简要介绍一下View的绘制流程，在我之前分析过的[Activity启动流程]()中有一步是ActivityThread执行`performLaunchActivity()`方法，这个方法内部首先会创建出Activity实例，然后调用Activity的`attach()`方法，在这个方法中创建Window（具体是PhoneWindow）。在`performLaunchActivity()`方法的最后会调用`callActivityOnPostCreate()`方法，之后Activity的`onCreate()`方法就会被调用。在`onCreate()`中调用`setContentView()`方法设置页面的布局，执行该方法最终会创建出页面的根布局DecorView。在这之后ActivityThread执行`handleResumeActivity()`方法，方法内部首先会调用`performResumeActivity()`方法，继而调用Activity的`onResume()`方法。`handleResumeActivity()`方法之后会调用WindowManager的`addView()`方法将DecorView添加到PhoneWindow中，通过一系列调用最终会执行ViewRootImpl的`performTraversals()`方法，在这个方法中会依次调用`performMeasure()`、`performLayout()`和`performDraw()`方法，通过这三个方法最终完成View的绘制（具体过程我就不说了）。上面我的描述可能不是很清楚，这里就简单总结一下整个流程的先后顺序：

* 1.创建Activity
* 2.创建Window
* 3.执行`onCreate()`
* 4.创建DecorView
* 5.执行`onResume()`
* 6.进行View的测量、布局和绘制

可以看出View的测量、布局和绘制都是在`onResume()`方法调用后才进行的，所以我们自然无法在`onCreate()`和`onResume()`中获取到View的宽高，我这里之所以要介绍View的绘制流程也是为了方便后面分析`View.post()`方法的原理。

好了，说回正题，我们在开发中可能使用过`View.post()`方法来在`onCreate()`方法中获取View的宽高，如下所示：

```java
view.post(new Runnable() {
    @Override
    public void run() {
        Log.e("TAG", "View的宽度为：" + view.getWidth() + " ,高度为：" + view.getHeight());
    }
});
```

那么为什么在`post()`方法中可以获取到View的宽高呢？我们来看看`post()`方法的定义：

```java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }

    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().post(action);
    return true;
}
```

可以看到这里首先判断了mAttachInfo是否为空，如果不为空就执行`mAttachInfo.mHandler.post()`方法，如果为空就执行`getRunQueue().post(action)`，那么这里究竟执行的是哪个判断呢，我先说一下结论，无论`post()`方法是在`onCreate()`还是`onResume()`中调用的，这里的mAttachInfo都为空，后面会分析到原因。我们直接来看`getRunQueue().post()`方法。

### 5.2.IdleHandler是什么



### 5.3.Handler的同步屏障机制



## 6.总结



## 7.参考文章

《Android开发艺术探索》

[一文看穿 Handler](http://yifeiyuan.me/blog/f77487d3.html)
