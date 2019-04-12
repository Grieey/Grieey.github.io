---
title : Review2总结
date: 2019-04-12T18:31:23+08:00
tags: 
    - 屏幕适配
    - Android
categories : 
    - Android
    - Review
---

### 设计稿和布局之间转换问题

不同的设备的分辨率不一致，所以设备的 **DPI** 不一致，实际显示时的像素计算公式为 **px = dp * density ** (其中density = dpi / 160),这导致了**dp** 转换为 设备上具体的像素值时，不同分辨率的设备不一致，最终的表现就是不同设备的布局不一致了。

在计算比例时，手机屏幕宽度的比例一般有比较稳定的数值项，所以在获取到设计稿的比例后，在代码中可以通过获取屏幕宽度来确定布局中控件的位置和大小。

在今日头条的方案中，是使用的 屏幕的总宽度(**px**) / 设计稿的总宽度(**dp**) 来计算**density** ，然后用这个 **density** 来乘以设计稿上**view的宽度** 就可以了，这样最后在设备上view 的整体占比和设计稿上view 的整体占比是一致的。



### 类的功能职责

#### startActivityForResult()

很多activity都有和其他activity交互后需要其返回一定的数据，这个时候应该清晰是谁来处理这些数据，比如有Activity **A** 需要传递数据到Activty **B** 进行处理，处理完成后返回到 **A** 时要告诉**A** 处理的情况(成功或者失败，类似)那么，这里的场景中，处理这些数据是 **B** 的职责，不管是 **A** 还是其他的activity，对 **B** 来说都是一样的流程，接收数据和返回对应的结果。所以接收这些数据和返回结果需要用到的键值对或者Code的维护工作都应该交给 **B** 来做，对于其他activity ，只需要知道他想传递数据并获取数据处理的结果，调用 **B** 中维护的方法即可。当功能结构或者代码发生改变时，无论是替换处理数据的 **B** 还是新增修改 **A** 类型的activity，另一方都无需改动，每次的修改都能保证到单一的维护，对应到处理数据这个功能是**B** 的职责应该它自己维护。(要是**AB** 都被替换了，那这个功能就是换血了，这种情况没有意义)

示例：

```kotlin
ActivityB：

  companion object {
    // 需要处理数据的键值对的key
    private var KEY_HANDLE = "key_handle"

    // forResult 需要requestCode 和 resultCode
    private const val REQUESTCODE = 33405
    private const val RESULTCODE_SUCCESS = 33406
    private const val RESULTCODE_FAILED = 33407

    // 普通的跳转
    fun startActivity(context: Context) {
      context.startActivity<ActivityB>()
    }

    // 需要传递数据并返回的跳转
    fun startActivityWithHandle(activity: Activity) {
      val intent = Intent(activity, ActivityB::class.java)
      intent.putExtra(KEY_HANDLE, true)
      activity.startActivityForResult(intent, REQUESTCODE)
    }

    // 是否来自该activity
    fun isResultFromActivityB(requestCode: Int): Boolean {
      return requestCode == REQUESTCODE
    }

    // 这里用于判断是否其他地方已维护了相同的code，用于判断数据处理是否成功
    fun isChanged(requestCode: Int, resultCode: Int): Boolean {
      return requestCode == REQUESTCODE && resultCode == RESULTCODE_SUCCESS
    }
  }

ActivityA:

	// 调用
	ActivityB.startActivityWithHandle(this)

	// 回调
	override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (ActivityB.isResultFromActivityB(requestCode)) {
      if (ActivityB.isChanged(requestCode, resultCode)) {
        // 业务逻辑
        ···
      }
    }
    ···
  }
```



#### 类的命名

将这个分类到类的功能职责是一个的类的命名应该和他的功能职责对应的，比如一个adapter，它是一个功能职责明确的adpater，如FoodAdapter，就是用来显示食物名称和图片的列表adapter，那它在业务场景中就应该只用来显示食物名称和图片；但是，这里可能有这种情况，因为FoodAdapter 本质上就是用来显示StringList的adapter，那为了方便，在需要显示人名的业务场景中也可以直接用这个adapter来完成业务逻辑，这样的代码功能没有毛病和问题，不过对项目来说就混乱了（当项目特别大时这种混乱带来的维护成本就很高了）。确实要复用，那么这个adapter就应该设计为可复用型的Commadapter ，如和FoodAdapter的区别就是里面的功能不仅仅是关于食物的了，而是一种共有属性的功能。使用者在用这个Commadapter时，就很清晰它是复用型的，而使用FoodAdapter也很清晰这个是和食物相关的adapter，单一的维护其功能职责。





