---
title: Review1总结kotlin中的关键字和操作符
tags: 
    - kotlin
  
categories : 
    - Android
    - Kotlin
---


### kotlin中的!!与?.

- **?.** 的理解：kotlin出现后一个非常亮眼的特点就是对空指针的处理，当使用的对象是可为空的对象时，调用其方法或者属性编译器会有`?.`和`!!.`的提醒 。从代码问题暴露出来感觉是，前者使用的场景是处理只需要判断是否为空而没有分支的情况，比如一个对象的添加，如果这个对象为空，那就不需要添加，这个时候可以使用`?.let{ }` 来优雅的处理为空的判断，而不需要写` if(xx != null) ...` 这种代码。

  但是如果说当前的业务场景有为空的分支需要处理，这个时候还是需要使用`if(xx != null)... else ...  ` 来处理，只是这个时候在不为空的分支中的对象，编译器会智能的将其可为空的对象转换为不可为空的对象，那就可以直接调用不用写`?.` 了

  示例:

  ```kotlin
  // 这里的bean可能为空，当为空的时候就不添加进去
  // 而不需要 if的判断了
  bean?.let{
    list.add(bean)
  }
  
  // 如果需要处理分支
  if(bean != null){
    // 在这个分支里，bean就会被只能转换为!,所以可以直接调用，而不需要bean?.xx()
    bean.xx()
  }else{
    // bean为null的业务处理
    ...
  }
  ```

  

- **!!** 的理解：个人觉得这个其实是在编译器层面就对编码者进行的提醒，那就是使用`!!`的地方，一定需要进行空判断的处理，很多时候写**Java**代码没有处理到空指针的情况就是因为忘了，而在kotlin中，可能为空的对象在使用时都有`!!`的提醒，这个时候就不会忘记去处理这里的空指针情况。**所以在所有使用`!!`的地方，必须进行空判断处理。** 

  示例：

  ```kotlin
  // 问题代码
  // 直接获取map中的value， value可能为空，这个时候null传入构造方法中就会引发异常
  fun getPlatHandler(context: Context, type: PlatformType): SSOHandler? {
      return when (type) {
        PlatformType.WEIXIN -> WXHandler(context, availablePlatMap[PlatformType.WEIXIN]?.platConfig!!)
        PlatformType.WEIXIN_CIRCLE -> WXHandler(context, availablePlatMap[PlatformType.WEIXIN]?.platConfig!!)
        PlatformType.QQ -> QQHandler(context, availablePlatMap[PlatformType.QQ]?.platConfig!!)
        PlatformType.QQ_ZONE -> QQHandler(context, availablePlatMap[PlatformType.QQ]?.platConfig!!)
        PlatformType.SINA_WEIBO -> SinaWBHandler(context, availablePlatMap[PlatformType.SINA_WEIBO]?.platConfig!!)
        else -> return null
      }
    }
  
  // 改进后
  // 因为这个方法本身就可能返回null,所以当发现map中的value为null时，使用 ?: 直接返回null
  fun getPlatHandler(context: Context, type: PlatformType): SSOHandler? {
      return when (type) {
        PlatformType.WEIXIN -> {
          val config = availablePlatMap[PlatformType.WEIXIN]?.platConfig ?: return null
          WXHandler(context, config)
        }
        PlatformType.WEIXIN_CIRCLE -> {
          val config = availablePlatMap[PlatformType.WEIXIN]?.platConfig ?: return null
          WXHandler(context, config)
        }
        PlatformType.QQ -> {
          val config = availablePlatMap[PlatformType.QQ]?.platConfig ?: return null
          QQHandler(context, config)
        }
        PlatformType.QQ_ZONE -> {
          val config = availablePlatMap[PlatformType.QQ]?.platConfig ?: return null
          QQHandler(context, config)
        }
        PlatformType.SINA_WEIBO -> {
          val config = availablePlatMap[PlatformType.SINA_WEIBO]?.platConfig ?: return null
          SinaWBHandler(context, config)
        }
        PlatformType.ALI -> {
          val config = availablePlatMap[PlatformType.ALI]?.platConfig ?: return null
          AliHandler(context, config)
        }
      }
  ```

  

### kotlin中的is和as

- **is** 的理解：这个有点类似于**Java** 中的**instanceof** 关键字，但是他做了更多的一些工作，如果我们对一个对象用`is` 进行了判断，在判断的分支中编译器会智能的将该对象转换为对应的类型。
- **as** 的理解：`as` 就是java中的强转，但是强转有时候会有失败的情况，所以在不确定类型的情况下，建议使用 `as!`