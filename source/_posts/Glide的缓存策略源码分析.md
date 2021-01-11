---
title: Glide的缓存策略源码分析
date: 2020-02-18T18:31:23+08:00
tags: 
    - Glide
    - Android
	- ReferenceQueue
categories : 
    - Android
    - Glide
    - 学习记录

---

> Glide在缓存的设计上分为了内存缓存和硬盘缓存两种。对于内存缓存，由于Glide使用的是Lru算法来作为缓存的策略，这种算法存在一个业务上的弊端就是，如果新增缓存时，正在使用的缓存被算法移除了怎么办呢？为了保证Glide加载的图片在使用过程中不会被移除了，Glide采用了弱引用+Lru的组合来保证。
>
> 文中源码版本：4.11.0

当我们调用`Glide.with(context).load(url).into(view)`时，`Glide` 内部会构建一个`Request`对象去具体执行`load`的逻辑，而在`Request`的内部则是由`Engine`去实际执行的`load`方法。

从下面的代码可以看到，先会从缓存中获取资源，如果不存在，才会新开线程从硬盘获取或者从其他远端（文件、网络等等）加载新资源。

![glide_engine_load](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_engine_load.png)

##内存缓存

内存缓存的设置也有两级，这里就是引言中使用的**弱引用+Lru**的方式：

![glide_engine_load_detail](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_engine_load_detail.png)

`activeResource`中的逻辑很简单清晰，内部维护了一个`HashMap`用键值对的形式保存了`key`和`WeakReference`，这里直接去获取，当存在就返回：

![glide_engine_load_weak](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_engine_load_weak.png)

如果弱引用中不存在了，就从`Lru`中去获取，从方法的调用上看也是一层层的获取，当缓存存在时，从`Lru`中取出来，并重新加入弱引用中。

![glide_engine_load_lru](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_engine_load_lru.png)

到这里为止，都是正常的流程。但我们知道，弱引用会在下一次`GC`时被回收，如果被回收了又没有加入`Lru`不就炸了么？对弱引用的使用不太了解的同学也许会存在这样的疑问，`Reference`有一个构造方法可以传入一个`ReferenceQueue`，这个队列的作用就是用于检测**对象**本身可达性发生变化时，通过Map来反向获取指向**对象**的包装引用（即`Reference`对象），将其放入到队列，来进行后续的操作，[这里有一篇关于引用队列的介绍](https://hongjiang.info/java-referencequeue/)。

正如上面的介绍一样，`Glide`的内部也是通过引用队列来实现**弱引用**对象被回收时，将其放入`Lru`中，看看具体的实现：

首先`ActiveResources`的构造方法中，会生成一个子线程执行检测任务，随时检测引用队列的`remove()`方法（这是一个阻塞方法，队列没有值的时候会一直阻塞），当还没有引用别回收时，这个检测的线程就会因为阻塞而休眠，而一旦出现了被回收的对象，就会被引用队列唤醒，执行`remove`方法的后续。

![glide_ar_init](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_ar_init.png)

在资源首次被添加时，会去生成一个弱引用的对象，然后在上面的检测中检测它的回收

![glide_ar_active](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_ar_active.png)

在`cleanupActiveReference`中，引用的资源还在时，就会新建一个`resource`然后通知`engine`把这个新的存到`lru`中，这样就完成了一次弱引用到`lru`的转换了

![glide_ar_turn2_lru](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_ar_turn2_lru.png)

这里还有一个细节就是每一次获取成功后都会调用`cached.acquire()`这个方法，这个方法内会自增一个计算的变量，用于统计资源的使用，每次`release`时都会自减，当这个值为0时说明没有使用他的地方了，同样的也会调用`listener.onResourceReleased`方法来将该变量存到`Lru`中。

## 硬盘缓存

硬盘缓存的使用是`DiskLruCache`，`Glide`也提供了几种缓存的策略如下

- DiskCacheStrategy.NONE： 表示不缓存任何内容。

- DiskCacheStrategy.SOURCE： 表示只缓存原始图片。

- DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。

- DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。

在最开始的图中，`waitForExistingOrStartNewJob`这个命名就基本说明了接下来的流程，先从本地的硬盘缓存中查找，如果没有就从远端资源获取。只不过这个地方的源码稍微有点绕。

`engine`在`waitForExistingOrStartNewJob`方法中构建了`DecodeJob`对象，并且开启了新的线程去执行它（这个对象实现了`runnable`）。

![glide_decode_job_run](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_decode_job_run.png)

根据上面不同中的`runReason`会生成不同的`stage`，然后生成不同的`generator`，主要有三种：

- ResourceCacheGenerator：这个类中会去寻找经过处理之后的资源
- DataCacheGenerator:这个类中会去寻找原始的资源
- SourceGenerator:这个类就是去远端获取资源

在`glide`可以同时缓存原始的资源和经过转换后的资源，当然这都是根据你设置的缓存策略来实现了。在`DecodeJob`中会去调用他们的`startNext()`方法，接下来就分别看看上面两个缓存的实现：

![glide_genertor_start](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_genertor_start.png)

这两个里面的实现大体逻辑都差不多，只是对于的`key`不一样，而这个`helper.getDiskCache`最后拿到的就是`DiskLruCacheWrapper`的实例，在他的`getDiskCache`中去获取磁盘的缓存：

![glide_disk_lru_cache](https://cdn.jsdelivr.net/gh/Grieey/ImgHosting@main/img/glide_disk_lru_cache.png)

到这里，会通过之前设置好的`callback`一步步调用到`engine`中，返回这个磁盘缓存的资源。