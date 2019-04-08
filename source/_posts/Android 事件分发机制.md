---
title: Android 事件分发机制
date: 2019-04-07T18:31:23+08:00
tag: 
    - Android
    - 事件分发机制
categories : 
    - Android	
    - 进阶
---



### Activity的事件处理

当事件产生时，事件首先会传递到Activity中进行处理，调用Activity的`dispatchTouchEvent` 方法，源码如下：

```java
		 /**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

- 在`Activity.dispatchTouchEvent` 中，如果是按下的操作，会调用`onUserInteraction()` 这个方法如下，它是一个空方法，实现这个方法可以知道用户通过设备主动拦截了事件，如当该activity 位于栈顶时，用户通过物理键触摸**back 、menu 、 home** 等操作都会回调这个方法。

  ```java
  		 /**
       * Called whenever a key, touch, or trackball event is dispatched to the
       * activity.  Implement this method if you wish to know that the user has
       * interacted with the device in some way while your activity is running.
       * This callback and {@link #onUserLeaveHint} are intended to help
       * activities manage status bar notifications intelligently; specifically,
       * for helping activities determine the proper time to cancel a notfication.
       *
       * <p>All calls to your activity's {@link #onUserLeaveHint} callback will
       * be accompanied by calls to {@link #onUserInteraction}.  This
       * ensures that your activity will be told of relevant user activity such
       * as pulling down the notification pane and touching an item there.
       *
       * <p>Note that this callback will be invoked for the touch down action
       * that begins a touch gesture, but may not be invoked for the touch-moved
       * and touch-up actions that follow.
       *
       * @see #onUserLeaveHint()
       */
      public void onUserInteraction() {
      }
  ```

- 第二句的`getWindow().superDispatchTouchEvent(ev)` 首先获取到**window类**的对象，而**window类 **是一个抽象类，**phoneWindow** 是 **window类** 的唯一实现类,因此这里直接看 **phoneWindow** 类的实现:

  ```java
   		// This is the top-level view of the window, containing the window decor.
      private DecorView mDecor;	
  
      ...
        
  		 @Override
      public boolean superDispatchTouchEvent(MotionEvent event) {
          return mDecor.superDispatchTouchEvent(event);
      }
  ```

  这里调用了 `mDecor.superDispatchTouchEvent(event);`  mDecor 是 **DecorView** 的一个实例，而**DecorView** 是phoneWindow类的一个内部类，是所有界面的父类，同时它继承自FrameLayout，而FrameLayout继承自ViewGroup。以下是**DecorView** 类的`dispatchTouchEvent`方法：

  ```java
  		public boolean superDispatchTouchEvent(MotionEvent event) {
          return super.dispatchTouchEvent(event);
      }
  ```

  这里开始其实就是一个事件向下的分发，如果最后这里有子view对该事件进行了消费，那么activity这里直接返回true，整个事件的流程便结束了，要是没有子view对这个事件消费，那么就调用activity的`onTouchEvent(ev);` 方法。

- 最后一句的`onTouchEvent(ev)` 调用的是activity本身的，该方法如下：

  ```java
  		/**
       * Called when a touch screen event was not handled by any of the views
       * under it.  This is most useful to process touch events that happen
       * outside of your window bounds, where there is no view to receive it.
       *
       * @param event The touch screen event being processed.
       *
       * @return Return true if you have consumed the event, false if you haven't.
       * The default implementation always returns false.
       */
      public boolean onTouchEvent(MotionEvent event) {
          if (mWindow.shouldCloseOnTouch(this, event)) {
              finish();
              return true;
          }
  
          return false;
      }
  ```

  当没有任何一个view处理时会调用这个方法，当return true的时候代表已经消费了这个事件，false 则没有，默认是总是返回false。这里首先调用了`mWindow.shouldCloseOnTouch(this, event)` ，代码如下：

  ```java
  public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
          final boolean isOutside =
                  event.getAction() == MotionEvent.ACTION_DOWN && isOutOfBounds(context, event)
                  || event.getAction() == MotionEvent.ACTION_OUTSIDE;
          if (mCloseOnTouchOutside && peekDecorView() != null && isOutside) {
              return true;
          }
          return false;
      }
  ```

  这里大概的意思就是如果事件在外界则消费事件，否则就返回false

至此，在activity 这一层的事件处理就分析完成了。