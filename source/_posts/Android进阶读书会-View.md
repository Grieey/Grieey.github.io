---
title: Android进阶读书会-View
date: 2019-07-07T09:31:23+08:00
tag: 
    - Android
    - textView
categories : 
    - Android	
    - 进阶	
    - View
---

> 最近在看皇叔的Android进阶，看完后还是结合实践来总结比较合适，就针对几个自己在业务中遇到的情景，结合书中的知识点进行实现。

## 自定义的NoPaddingTextView

Android开发在实现UI图时，总会遇到Android的textView存在内边距的情况，应对普通的文本对齐设置个大概倒是没什么毛病，但是有些情况还是无法处理，比如要求文字的底部和图片对齐，因为不同的手机，不同的字体大小等等都会影响到内边距的大小，而且因为是对齐图片，所以没法设置个大概，这里能考虑到的就是自己去实现一个没有内边距的textView

### 实现思路

- 获取到需要绘制的内容，并通过paint对象来计算绘制内容的高度和宽度

- 在`onMeasure`方法中，根据绘制的内容所需要的最小的宽度和高度设置View的宽高
- 重写`onDraw`方法，绘制内容

接下来就是一步一步去实现思路中的步骤

### 获取文本的宽高

1. 这里涉及到关于**TextView** 的一些知识点，在TextView中，文字的绘制基本是按照下图的中线来进行的

![图1](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/textView.jpeg)

- **top、bottom**这两个值的绝对值之和是文字绘制中允许的最大高度（这个值取整后为textView的高度），其中**top** 的值是**top到baseline**的相对位移大小，为**负值** ;**bottom** 为**bottom到baseline**的相对位移大小，为**正值**。

- **ascent、descent** 这两个值为绘制普通文字的时候，顶部和底部的范围，他们和`getFontSpacing()`方法获取的数据是一样的，同样的**ascent为负值、descent为正值**。和**top、bottom**的区别是，某些特殊的字符在绘制时可能超过**ascent、descent**，但是不会超过**top、bottom**,因此后者的值一般是大于前者的。

- **baseline**,文本绘制时的基准线，就是一条文本的重心线，这条线会使得整个文本看起来比较和谐。同时，这条线也是`canvas.drawText()`方法中，**参数y**的起点值，也就是在绘制文本时，绘制的起点是以**baseline**的位置不是其他**canvas**绘制时的左上角**(0,0)**

  其他更多的关于textView的知识点可以看这篇文章[HenCoder Android 开发进阶：自定义 View 1-3 drawText() 文字的绘制](<https://hencoder.com/ui-1-3/>)

2. 第二点就是关于**paint**的`getTextBounds()`方法，这个方法获取的是文本的显示区域，也就是文本本身的大小，这个和上面绘制的各种线的区别是，上面各种线可以看做是为了适配不同的文字在绘制时的美观和整齐所作出的一些标准规则，而文本显示区域就是实际上文字绘制后显示的大小。

   下图是文本绘制时的一些数据

   ![图2](https://raw.githubusercontent.com/Grieey/Grieey.github.io/source/img/textViewValue.jpeg)

   其中rect就是传入`getTextBounds()`方法后获取到的文本显示区域的矩形大小，这个值比上面的**ascent、descent**之和还要小。

3. 有了以上两点的知识后，在获取文本的宽高时就比较清晰，要清除内边距，那就是和UI设计稿一样的显示，文字是多大显示的就是多大的大小，所以通过`getTextBounds()`来获取显示文本的矩阵大小

###设置View的宽高

在重写的`onMeasure()`方法中，会传入**widthMeasureSpec、heightMeasureSpec** 这两个值包含了宽度和高度的测量方法和大小，所以在设置的时候先对其测量方法进行判读，如果是精确模式测量的数据，就不需要再赋值处理，如果不是，就需要我们动态的根据绘制内容计算出需要的大小。代码如下

```kotlin
private fun getViewWidthAndHeight(widthSpec: Int, heightSpec: Int): Pair<Int, Int> {
    val widthMode = MeasureSpec.getMode(widthSpec)
    val heightMode = MeasureSpec.getMode(heightSpec)
    val width = MeasureSpec.getSize(widthSpec)
    val height = MeasureSpec.getSize(heightSpec)

    val realWidth = if (widthMode == MeasureSpec.EXACTLY) {
      width
    } else {
      rect.right - rect.left
    }
    val realHeight = if (heightMode == MeasureSpec.EXACTLY) {
      height
    } else {
      layout.lineCount * (rect.bottom - rect.top) +
          ((layout.lineCount - 1) * lineSpacingExtra * lineSpacingMultiplier).toInt()
    }

    return Pair(realWidth, realHeight)
  }
```

最后再`onMeasure()`中调用`setMeasuredDimension()`方法重新设置好View的宽高

### 绘制内容

先获取到需要绘制的字符串，在有多行文本时，由于直接绘制，并不能实现换行的效果。所以讲文本内容的每一行添加到一个数组中，每次绘制一行的数据

```kotlin
private fun getTextContents() {
    lineContent.clear()
    for (i in 0 until layout.lineCount) {
      val start = layout.getLineStart(i)
      val end = layout.getLineEnd(i)
      lineContent.add(text.toString().substring(start, end))
    }
  }

···

override fun onDraw(canvas: Canvas) {
    lineContent.forEachIndexed { index, it ->
      val space = index * (lineSpacingExtra * lineSpacingMultiplier + rect.bottom - rect.top)
      canvas.drawText(it, -rect.left.toFloat(), (-rect.top) + space, textPaint)
    }
  }
```