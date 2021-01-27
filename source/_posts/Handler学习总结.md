---
title: Handler学习总结
date: 2021-01-26T18:31:23+08:00
tags: 
    - Handler
    - Android
    - 源码分析
categories : 
    - Android
    - 学习记录
---

> Handler 是Android 中线程间通信的一个工具。在源码中许多涉及线程间通信的场景都是使用Handler来进行通信的，本篇为从源码上学习Handler整个运行机制的总结。

## 前言

我们都知道**Android**是一个消息驱动的系统，当我们的应用在运行后，就会源源不断的处理各种消息，以此来保证程序的正常运行。其实最开始接触到这个概念，是因为一道面试题，问`Looper`中有一个死循环，为什么我们的程序不会被卡死，这个时候一搜网上的很多回答都是因为**Android**是一个消息驱动型的系统。确实，这解决了为什么这里是个死循环的问题，因为它需要不断处理消息；但是自己下来仔细一想，还是不明白这个消息型驱动是个什么概念，准确来说是，不理解这里为什么需要使用一个死循环来实现这种机制。

那么回想一下我们最开始学习**Java**时，就是编写一段有输入输出的GUI程序：写完后运行起来->我们输入数据->程序处理输入数据->输出结果->程序结束。这是最简单的一个程序，他在处理完成我们的任务后，是会结束的！而且刚开始学习的时候，老师也会告诉我们，写循环的时候需要注意循环的条件，不要写死循环，这样程序就会卡死在那儿。那个时候其实不太理解这种卡死真正是什么，感觉像刚接触电脑时那种点哪儿都没有反应，现在回想起来，那种卡死是假象，并不是生活中的某个东西卡住了不动了。而且程序一直在运行，运行着你的循环中的代码。所以，当你的循环的代码能够不停的处理不一样的事件，那样不就能一直处理新的需求了。如果没有循环，单线程上的代码执行完，程序就结束了。

所以说循环是让程序能一直运行的一个基础，但是对于单线程来说，循环里的东西是固定的，那这样只是在不断的处理同一个东西而已，并没有什么意义。所以肯定需要多个线程来产生不同的东西，然后交互信息，这样才能源源不断的处理新的东西，我们把这个信息封装一下就可以称为消息了，而多个线程交互这个信息其实就是线程间通信了，这个通信的基础就是多个线程都能访问到同一资源（这个资源就是通信的媒介）。

这样一想，这个工作模型就是**生成者-消费者模型**呀。以工厂为例子，生产者就是生产零件的，消费者就是组装零件的。我们可以把应用看着一个消费者，`Looper`的死循环就是传送带。这样，生产者可以是系统，产生新的刷新信号，也可以是用户，触摸屏幕产生新的事件，他们在不同的传输带工作（线程），当他们完成他们的零件生产后，就将零件（事件）包起来（封装到消息）中，通过工人（`Handler`）将消息放到传送带上。这样就完成了两个传送带的通信了对不对，生产的传送带将某一批号的零件生产成功的信息通过包装放到另一个传送带上，那么另一个传送带处理该包装时，就知道了该批号零件生产成功的信息，接着按照包装上的说明书去组装他们（处理消息）。

接着我们就从源码的角度来看看**Android**中整个机制是如何具体运行的。

## 整体流程

![handler_flow3](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_flow3.png)

图中涉及到的一些对象：

- **Handler：**本质上是作为一个工具类，暴露API给外部调用，其内部的消息管理都是交给了**Looper**去具体执行。作为开发者只需要关注使用`handler`对象去发送消息和处理消息。
- **Looper(蓝色框中的)：**具体做消息的分发处理的`工具人`，作为**Java**层中的话事人去，连接着消息的发送者和处理者**Handler**、消息队列的维护者**MessageQueue**及线程的信息。一个线程中只有一个`Looper`对象存在。
- **MessageQueue：**维护消息的队列，同时还负责与**Native**层进行通信。
- **Message：**消息的抽象类，内部存储着具体的消息内容，消息的类型（同步或者异步），消息的时间和Callback、handler对象等等。
- **NativeMessageQueue：Native**层中的消息管理者的角色，承担了类似**Java**层中的handler部分职责，**Java**层可以通过**Jni**调用到该对象中的方法。在**Native**层中也存在**Looper(绿色框中的)**和handler对象，用于处理**Native**层的消息。
- **MessageHandler：Native**层中的消息处理者。
- **epoll：**linux中的IO多路复用机制，性能优于select机制和poll机制。

## 源码分析

### 初始化

这里我先不按照流程来看代码，从开发者使用的角度来分析源码，首先接触的是`handler`的初始化：

[frameworks/base/core/java/android/os/Handler.java]

![handler_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_init.png)

在`Handler`的构造方法中，调用了`Looper.myLooper()`方法获取`Looper`对象赋值给`Handler`，从这里可以看出，在线程中使用`Handler`需要先初始化好`Looper`对象，因为`Handler`本身就是为了线程通信而服务的，如果线程相关的信息都没有准备好，这服务也就没有意义了。

[frameworks/base/core/java/android/os/Looper.java]

![handler_get_looper](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_get_looper.png)

继续跟代码来看看这个`Looper.myLooper()`干了啥，从上面代码看，就是从`sThreadLocal`中获取`Looper`对象实例。这个`sThreadLocal`的类型为`ThreadLocal<Looper>`对象，他是**JMM**中每个线程的私有内存所存储的东西的具体实现（`ThreadLocalMap`，一种类似**Map**的结构，以`Thread`为**Key**，存储对应的**Value**），[关于ThreadLocal的传送门有更多了解可以去这里看看](https://juejin.cn/post/6919002656186826765)。因此每个`Handler`内的消息队列所维护的都是本线程中的所有消息。

这个`sThreadLocal`的上面还有一句注释，就是说在调用`get()`去获取`Looper`对象时，应该先调用过`prepare()`这个方法，去具体看看：

![handler_looper_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_looper_init.png)

在这个方法里，主要做了三件事：

- 调用`Looper`的构造方法生成一个实例。
- 在`Looper`的构造方法中，新建了`messageQueue`对象，并绑定了线程。
- 将实例设置到`threadLocal`中。

这样，对`Looper`来说它的初始化工作算完成了。上面的代码还有一个细节就是`sThreadLocal.get() != null`的判断，也说明了对于一个`Looper`来说，只能调用一次`prepare()`方法。

接着是`MessageQueue`的初始化，主要是通过**Jni**调用了一个**Native**方法。

![handler_message_quque_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_message_quque_init.png)

在**Native**中调用到的是`android_os_MessageQueue.cpp`中的方法

[frameworks/base/core/jni/android_os_MessageQueue.cpp]

![handler_message_quque_native_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_message_quque_native_init.png)

`NativeMessageQueue`的构造方法主要是设置`NativeLooper`对象（为了和**Java**层的`Looper`做一下区分，这里加一个Native的前缀）。感觉这里的`NativeMessageQueue`有点像`Handler`的意思。

![handler_looper_native_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_looper_native_init.png)

在`NativeLooper`的初始化中主要是做了`Epoll`初始化相关的操作。创建了`Epoll`的实例，并注册了唤醒事件和其他事件到`Epoll`中。这里涉及到的是`epoll_create`和`epoll_ctl`两个函数，前者会创建一个空闲的`fd`并将他们关联在一起；后者可以注册一些事件进行监听。

这里简单说一下，对于`epoll`的使用基本上分为三步：

1. 通过`epoll_create`创建并关联空闲`fd`；
2. 通过`epoll_ctl`添加我们想要监听的事件到`epoll`中；
3. 通过`epoll_wait`去等待我们监听的事件发生。

在整个初始化过程中，前面两步已经准备好了，在下面的流程中就会用到第三步。`Epoll`机制涉及到就是**Linux**内核相关的一些知识了，可以看看袁辉辉大佬写的[源码解读epoll内核机制](http://gityuan.com/2019/01/06/linux-epoll/)。有能力的小伙伴也可以直接阅读源码去分析它的工作机制[epoll源码](https://github.com/torvalds/linux/blob/master/fs/eventpoll.c)

`Epoll`在这里的作用就是**生产者-消费者模型**中的阻塞机制，当生产者没有生产出消息时，消费者（线程）就需要调用`epoll_wait`等待；消费者发出消息后，可以通过写入唤醒事件，在`Epoll`中监听到写入事件后，`epoll_wait`方法就会返回事件给消费者-线程，也就可以继续执行消息的处理逻辑，然后再循环到`epoll_wait`进行等待下一条消息。

至此，整个准备工作就完成了，接着就是运行起来整个机制。调用`Looper.loop()`方法，该方法就是前面介绍的死循环，所以我们的所有事件和消息处理都在这个循环中。

### 发送消息

当我们在线程中拿到`Handler`的实例时，可以调用一系列方法去发送消息，不管是发送我们自己`obtain()`的消息还是`runnable`，到`Handler`内部都会封装为一个`Message`进行传递，`runnable`会设置到`message`的`callback`中。

而在`Handler`内部，一系列方法，最后都是调用的`sendMessageAtTime`，而`sendMessageAtTime`方法调用了`enqueueMessage`。

![handler_enqueue_message](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_enqueue_message.png)

这一步中，将`message`的几个关键信息进行了设置，最重要的就是`target = this`，将当前的`handler`对象设置为了消息的`target`，这个作用是以便后面取出消息的时候，知道找谁去处理他。虽然一个线程中`Looper`只有一个实例，但是我们可以创建多个`Hanlder`的实例来进行消息的发送和处理，所以这个的目的是将发送方和处理方通过消息关联。

继续跟代码到`MessageQueue。enqueueMessage()`

[frameworks/base/core/java/android/os/MessageQueue.java]

![handler_queue_en_message](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_queue_en_message.png)

这一个方法中的逻辑主要是根据消息时间将消息插入到合适的位置，同时根据消息队列的情况看是否需要唤醒线程。

对于唤醒的逻辑，我是这样理解的：

- 第一种情况是如果当前队列没有消息，那么新消息就是队头，这个时候是否需要唤醒当前线程需要根据`mBlocked`的状态（这个状态的设置可能是之前的消息队列为空了，所以没有消息处理，就被**Native**阻塞了）；
- 第二种情况就是看是否满足需要处理异步消息的判断，也就是存在同步屏障并且当前消息是异步消息时，如果当前仍然是阻塞状态，那么就需要立刻唤醒线程来处理这条优先级高的异步消息了。

额外的一个逻辑：这个循环里面有个将`needWake`置为`false`的判断，这个判断中如果`needWake`为`true`了，说明前面的`mBlocked && p.target == null && msg.isAsynchronous()`是`true`，而且执行到这一步了，也说明了现在的队列不为null了，再看第二个条件是`p.isSynchronous()`，综合一下就是之前已经存在需要唤醒线程的异步消息了（也就是这条`p.isAsynchronous()`的消息也经过`mBlocked && p.target == null && msg.isAsynchronous()`的判断，但到当前消息判断时，线程仍然为`Blocked`状态，说明上一条的唤醒还没有成功，处于唤醒的过程中），所以需要重置`needWake`来避免重复唤醒。

[frameworks/base/core/jni/android_os_MessageQueue.cpp]

![handler_queue_wake](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_queue_wake.png)

最后看看唤醒线程的代码，这里会通过`mWakeEventFd`写入一个8字节的数据，这个时候**epoll_wait**会收到写入事件，然后返回值给`nativie`中`Looper`循环，就可以继续遍历消息了，这个逻辑在接下来的分发消息中就会涉及到了。`TEMP_FAILURE_RETRY`是一个重复尝试，直到成功的方法。

### 分发消息

分发处理消息主要是`Looper.loop()`方法里的逻辑

![handler_looper_loop](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_looper_loop.png)

`Loop()`方法的逻辑比较简单，调用`queue.next()`去获取下一条消息，然后调用`msg.target.dispatchMessage(msg)`去处理它。

![handler_queue_next](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_queue_next.png)

`next()`方法中的逻辑比较长，大致可以分为三个部分：调用到**Native**层、处理同步异步消息和执行`idle`任务。上面的代码是一个总览，可以从头到尾的看一下整个流程。下面就分别说说三个部分的具体逻辑。

首先看调用到**Native**的处理。`nativePollOnce()`方法，传入了`mPtr`和`nextPollTimeoutMillis`这两个值，前一个是`nativeMessageQueue`的地址，在`messageQueue`构造函数中从**Native**返回过来的，这里传回去方便操作**Native**对象，后面一个`nextPollTimeoutMillis`就是**Java**层需要的延时唤醒时间了，这个值有值说明**Java**层的消息需要延迟处理。该方法在`native`中直接调用到了`nativeMessageQueue`的`pollOnce`，进一步调用了`mLooper`的`pollOnce`。我们就直接来看`mLooper`的源码。

![handler_native_looper_pollOnce](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_native_looper_pollOnce.png)

这个方法里也是一个死循环去处理消息，前面遍历`response`是处理当前接收到的非唤醒时间的请求，这个可以在`polllInner`方法的逻辑中看到，当`epoll_wait`接收到的不是唤醒事件，而是其他事件时，会先将事件处理并添加到`mResponse`数组中去，这个时候，当新的循环遍历到来就可以去处理它们了。

接着是进入到`pollInner`中：

![handler_native_looper_inner](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_native_looper_inner.png)

这个方法的逻辑也可以分为三个部分：

- 调用`epoll_wait`方法阻塞等待事件的发生，这里可能发生多种事件，如果是唤醒事件，则对应前面新消息的处理或者延迟消息的处理，这个时候会清空管道内的消息，同时执行下面两个部分的事件处理。
- 遍历`mMessageEnvelope`中的消息，调用`messageHandler`来处理消息。
- 执行`mResponse`中带`callback`的事件。

如果后面两种逻辑没有进行事件的处理，那么这个时候`pollOnce`的死循环就会继续进行到`pollInner`中，再次执行`epoll_wait`去等待唤醒事件。

如果后面两种逻辑进行了事件的处理，这个时候**Native**的死循环就会结束，对**Java**层来说这一次的阻塞就结束了，**Java**层的循环逻辑就会继续走。

回到**Java**这边，当**Native**的阻塞结束后，就开始处理**Java**中的消息了。主要是同步和异步消息的处理：

![handle_queue_find_msg](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handle_queue_find_msg.png)

当存在同步屏障时`target == null`就会仅处理异步消息，否则就按照顺序去处理同步消息。这里会处理消息的时间，要是大于了当前时间，计算出`timeout`时间，在下一次遍历时会通过`nativePollOnce`出给**Native**去阻塞，在时间到了后唤醒自己来处理这条消息。

当然，要是这个时候仍然没有消息可以处理，那我们也可以不让**CPU**闲着，这个时候会进行`idle`的任务处理

![handler_queue_handle_idle](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_queue_handle_idle.png)

`idle`的逻辑就比较简单了，从`mIdleHandlers`中最多取出4个任务来执行，执行一次后本次`next()`方法的循环就不会再执行`idle`的任务了，需要等到下一次`Loop()`方法调用`next()`的时候才会有可能再去执行`idle`。这么看其实执行`idle`的优先级是最低的，因此我们可以把一些不重要的工作放在`idle`中执行，而且不能影响到正常消息的处理。

### 处理消息

![handler_dispatch_msg](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_dispatch_msg.png)

在`Looper`的处理中，会调用`msg.target.dispatchMessage`；这个方法中，可以看到有限处理的是消息的`callback`，也就是我们调用`handler.post`时发送的那些`runnable`。接着才是处理`handler`本身的`mCallback`，最后才是处理消息。由此可以看出`handler`在处理的时的优先级，又因为**Android**中很多通信都是使用的`handler`，也许解决某些业务的时候可以从这个点入手。

### 小结一下



## 扩展思考

### 关于Handler中的内存泄漏

内存泄漏是个经常讨论的话题，我们这里仅探讨下关于`Handler`中的内存泄漏问题。无论是以前使用`AsyncTask`还是现在在主线程中使用`Handler`进行通信，网上都有说需要注意`Handler`的内存泄漏问题。Android中的内存资源的重复利用是通过`GC`机制来保证的，当我们不再使用的内存，会通过`GC`来回收，不过这个回收的条件是有一定规则的，就是`GC`需要判断这块内存是否和`GCRoot`相连接，如果连着，说明不能回收。（为什么这么设计具体可以看《深入理解Java虚拟机》这本书，有非常详细的`GC`机制介绍）。而发生内存泄漏的原因也是因为本来应该回收的内存仍然和`GCRoot`连着，能作为`GCRoot`的对象有好多类型，其中一个就是`活跃的线程`。这就和`Handler`的使用对应上了，`Handler`用于线程间通信，回顾下上面的流程，`Handler`创建于`A`线程的`ActivityA`页面，然后`B`线程拿到`handler`实例，发送一个消息，这个时候在`A`线程的`MessageQueue`中就存在一条消息，它的`target`为`ActivityA`中的`handler`对象，我们一般使用`handler`时，都会用匿名内部类实现`Callback`接口来处理消息，而在**Java**中，匿名内部类是会隐式的持有外部类的引用的。

将上面的一系列引用串在一起就是，**GCRoot(UiThread)->MessageQueue->Message->handler->Callback->ActivityA**

当`ActivityA`在`onDestory()`之后本来应该会被回收的，但是由于某些情况下，消息没有来得及处理，以至于上面的引用链还存在，这个时候，他就没有办法回收了。

所以有解决方案是，在`ondestory()`时，调用`handler.removeMessages()`去清除队列中的消息，这样上面的引用链就断开了，也就解决了内存泄漏问题。

还有之前的方案是使用静态内部类和弱引用，因为静态内部类不会隐式的持有外部类的引用，这样也可以去避免这个问题，不过从上面引用的理解上来说，这种思路不算是一种治本思路。

### 同步屏障的使用

同步屏障可以保证一些优先级比较高的消息被优先处理。它目前有一个应用的地方是在`ViewRootImpl`中，当`Sync`信号到来时，会触发`ViewRootImpl`中的`scheduleTraversals()`方法，代码如下

![handler_sync_do](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_sync_do.png)

上面是`mChoreographer.postCallback`方法发布了`mTraversalRunnable`，这个方法内部层层调用后其实也是将消息插入到主线程的`MessageQueue`中：

![handler_msg_do_schedule](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/handler_msg_do_schedule.png)

最后在主线程执行绘制，这样能保证绘制相关的消息被优先处理。