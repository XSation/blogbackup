---
title: Android开发艺术（3）——View的事件体系
date: 2017-09-19 21:12:22
tags:
categories: 阅读笔记
---

# View的基础

## View的位置参数

表示位置的几种参数

- left/right/top/bottom表示view（view最初的状态）距离ViewGroup的左上右下的距离
- 3.0之后：x/y 表示view（view的当前状态，平移后状态会变）的左上角的坐标，也是相对于ViewGroup的
- 3.0之后：translationX/translationY，表示view（view的当前状态，平移后状态会变）相对于ViewGroup的偏移量

```java
width = right - left;
x = left + translationX;
```

**注意**：View的平移其实只是改变了x/y和translationX/translationY，left/right/top/bottom还是不变。

x改变，如果left不变，translationX也会改变

## MotionEvent和TouchSlop

- MotionEvent
  - 手指触摸（down、move、up等）之后，会返回一个MotionEvent对象，通过它，可以拿到x、y（表示当前手指位置相对于被作用的控件——View或者Activity的坐标），rawX、rawY（表示当前手指位置相对于屏幕坐标）
- TouchSlop
  - 这是一个常量，根据设备不同而不同，系统推荐我们利用它作为一个临界点，如果手指滑动小于这个值，就认为没有滑动，这样可以提升用户体验，当然不用他来处理也无所谓(这一点可以在demo中体现，即使两次返回的MotionEvent的坐标差小于TouchSlop也是可以的)
  - ```ViewConfiguration.get(context).getScaledTouchSlop()```

## VelocityTracker、GestureDetector和Scroller

### VelocityTracker

> 可以称其为速度追踪器，就是获取手指滑动的速度的。当然在手机上，指的是一定时间内，手指划过的像素

#### 使用：

1. 首先需要在onTouchEvent中调用它，这样才能采集到手指在屏幕上的坐标变化信息
2. 获取速度前需要先计算，传入时间，表示需要计算的时间长度
3. 获取2传入的时长内滑过的像素，即使速度，不过这个速度的定义跟以往不同，只是像素长度而已
4. 使用完后要释放

#### 代码：

```java
//初始化
    VelocityTracker velocityTracker = VelocityTracker.obtain();
//添加MotionEvent(在onTouchEvent中)
    velocityTracker.addMovement(event);
//计算速度 
	velocityTracker.computeCurrentVelocity(30);
//获取速度
    velocityTracker.getXVelocity();
//释放
	velocityTracker.clear();
	velocityTracker.recycle();
```

### GestureDetector

手势监控，使用方式十分简单

```
//1.创建GestureDetector，注意只能在Looper Thread中使用，看源码可知，内部使用了handler
gestureDetector = new GestureDetector(Context,GestureDetector.OnGestureListener);
gestureDetector.setIsLongpressEnabled(false);//是否开启长按，如果开了，可以捕获到长按事件，但是长按后不可以捕获到滑动事件
//2.在view的onThouchEvent中让它接管事件，然后根据他的返回值决定是否消费(return true)
boolean resume = gestureDetector.onTouchEvent(event);
return resume;
//3.在OnGestureListener接口的各种方法就会被调用
```

这个类只是辅助类，帮助我们更容易的捕获到手势（双击、长按等，当然也可以自己写，也就是各种onTouchEvent中的处理）

### Scroller

> 注意，Scroller实质是不断的调用scrollTo方法，所以就要了解scrollTo方法

scrollTo移动的是View的内容，所以，Scroller会给View的内容增加滚动效果

#### 使用

```java
//1.创建Scroller
scroller = new Scroller(context);
//2.重写View(需要滚动的东西所在的View，因为scrollTo滚动的是View的内容)的computeScroll方法 
public void computeScroll() {
    if (scroller.computeScrollOffset()) {//该次滚动是否执行完
	scrollTo(scroller.getCurrX(),scroller.getCurrY());
	//postInvalidate();//这里书上写错了，不需要调用他，因为scrollTo之后就会自动重绘了
	}
}
//3.调用Scroller的startScroller方法
scroller.startScroll(0,0,-10,-10,1000);
invalidate();//注意调用之后还需要调用这个方法，让view重绘，具体原因之后Scroller原理的方法
```

# View的滑动

View滑动的三种方法：

- 通过View的scrollBy、scrollTo（内容滚动）
- 通过动画
  - 补间动画（只是改变的影像）
  - 属性动画（3.0以上才行，3.0以下可以使用兼容库实现，但是本质还是**补间动画**。sh属性动画改变的就是属性了——translationX）
- LayoutParams（得看你改变的是什么left、x、translationX都可以，效果不同）

## ScrollTo/ScrollBy

scrollBy内部调用的是scrollTo，scrollTo其实就是修改mScrollX、mScrollY的值，然后调用invalidateParentCaches实现View的内容的位置改变

mScrollX：View的左边缘距离内容的左边缘的距离，内容边缘在右边，mScrollX为负，反之为正（正好相反）	

## 三种对比

- scrollTo/scrollBy：操作简单，适合对View的内容滑动
- 动画：操作简单，可实现复杂动画效果
- layoutparams：操作复杂，可以满足各种需求

# 弹性滑动

## Scroller

原理：

- 当调用scroller.startScroll（）的时候，内部只是保存了一些参数，比如结束的位置，动画时长等

- 然后调用invalidate（），此时会导致view重绘，然后在```boolean draw(Canvas canvas, ViewGroup parent, long drawingTime)```方法中就会调用```computeScroll```方法

- 我们重写了```computeScroll```，首先调用```scroller.computeScrollOffset()```，它内部就会计算出接下来要移动的scrollX，是否scroll完成等，如果返回true，就从scroller中拿到scrollX，调用scrollTo，然后就会自动重绘，（书上还调用了postInvalidate，其实没有必要，scrollTo之后就已经重绘了），让View重绘，这样就又回到了第二步

  > scrollTo，会导致view重绘，试验一下，确实是这样的，去掉postInvalidate也没问题，看来书上写的确实不对

  #### 重点

  Scroller的精髓在```scroller.computeScrollOffset```

  ```
  public boolean computeScrollOffset() {
          if (mFinished) {//如果动画结束了，直接return false
              return false;
          }

          int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);//已经执行了动画的时间
      
          if (timePassed < mDuration) {//如果小于动画时长，就往下走，否则把mCurrX置为最终的X，并把mFinished置为true。不得不说，太严谨了，mCurrX重新赋值，仔细想想。。。
              switch (mMode) {
              case SCROLL_MODE://并没有做什么，只是根据当前进度(动画总时长1秒，过去了0.5秒，进度就是50%)，计算出mCurrX
                  final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                  mCurrX = mStartX + Math.round(x * mDeltaX);
                  mCurrY = mStartY + Math.round(x * mDeltaY);
                  break;
              case FLING_MODE://这种模式比较复杂，先不管了
  				。。。
                  break;
              }
          }else {
              mCurrX = mFinalX;
              mCurrY = mFinalY;
              mFinished = true;
          }
          return true;
      }
  ```

  > scroller对view没有引用，也没有用计时器，只是通过当前时间和初始时间，计算出已经执行的时间，然后根据动画所需要的总时间，计算出当前动画的执行进度，很棒的思路。。。

  ## 通过动画

  没什么好说的

  ## 延时策略

  handler发消息、thread.sleep、view.postDelay等，随意发挥

  # View的事件分发机制

>  以前是看博客，这次自己来搞，下载2.3的源码（先用的7.0、4.0的源码，实在看不懂，2.3确实简单点了），搞起

可以对着源码来看

### ViewGroup:

```
public boolean dispatchTouchEvent(MotionEvent ev) {
	//过滤一些“错误”的事件，直接return false,
	if (!onFilterTouchEventForSecurity(ev)) {
            return false;
    }
    
  	//FLAG_DISALLOW_INTERCEPT(一个标记，表示是否需要拦截事件，
  	//这个是由childView来设置，通过它可以使chilidView拥有控制parentView
  	//是否拦截事件的权利。7.0的源码中，这个值会在ACTION_DOWN的时候重置，2.3
  	//的源码中暂未找到。这个标记一般在onTouchEvent的ACTION_DOWN之后的事件
  	//中设置，所以即使重置也无影响)
    boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

	//针对down做一些处理
    if (action == MotionEvent.ACTION_DOWN) {
    	if (mMotionTarget != null) {
    			// mMotionTarget是事件作用的View，down的时候，应该还没有它，
    			// 这里不为空，所以是特殊情况，直接给他置空
                // this is weird, we got a pen down, but we thought it was
                // already down!
                // XXX: We should probably send an ACTION_UP to the current
                // target.
                mMotionTarget = null;
    	}
    	//childView不让parentView拦截  或者 自己的onInterceptTouchEvent返回false
    	if(禁止拦截 或 !onInterceptTouchEvent(ev)){
        	//不拦截，那就找合适的childView(位置之类的满足条件的)，把事件分发给他们
        	for(遍历childView){
            	if(childView可接受事件——处于事件所在的坐标等条件满足){
            		//分发给childView(childView可以为viewgroup也可以为View)
            		//如果是viewGroup，就和现在分析的这套代码相同，否则稍后分析
                	if (child.dispatchTouchEvent(ev))  {
                		//这个孩子处理了事件了(可能是他自己处理的，也可能是他的孩子处理的)
						// Event handled, we have a target now.
						//给mMotionTarget赋值，表示事件将作用于它(这个它其实是与其他的子View)
						//做区分的，因为事件可能最终被它的几重孙子消费，但一定是这个孩子的子孙
						//不会是它的兄弟们
                        mMotionTarget = child;
                        //既然处理了，作为父亲，也return true，跟上行注释类似，他也告诉
                        //他的父亲，他处理了事件了(其实这里是他的孩子处理的)
						return true;
					}
					//这个孩子（或者孩子的孩子）没处理，那就分发给下一个孩子，只要有一个
					//后代处理了，上面就会return true，当前我们所分析的这个View的任务就完成了
            	}
        	}
        	//所有的孩子、孙子都没有处理，自己来处理
    	} 
    }
    
    //下面这两行代码&看不太懂，反正就是改变了mGroupFlags，而mGroupFlags在前面获取disallowIntercept
    //的时候用到过，再结合下面那个Note，再结合7.0源码中ACTION_DOWN后会重置FLAG_DISALLOW_INTERCEPT，
    //所以这里的意思大致应该是：如果UP活着CANCEL,就设置mGroupFlags，这将导致下次DOWN后
    //FLAG_DISALLOW_INTERCEPT变为“flase”，功能类似于7.0的重置
    //(*^__^*) 爽啊，从源码中找到答案的感觉真爽。。。
    boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||
                (action == MotionEvent.ACTION_CANCEL);
	if (isUpOrCancel) {
		// Note, we've already copied the previous state to our local
		// variable, so this takes effect on the next event
		mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
	}


	//能走到这里有以下几种情况：
	//1.是DOWN且target==null(是DOWN且child不处理)
	//2.不是DOWN且target==null（自己处理）
	//3.不是DOWN,target！=null(DOWN已经由child处理，这个事件不是DOWN，孩子、自己都可以处理)
	

	//如果target==null，说明没有childView要处理事件，交由自己处理
	//如果target！=null，说明这个不是ACTION_DOWN且DOWN已经被child处理，这是一个非DOWN
	final View target = mMotionTarget;
	if(target == null){//没有childView处理
    	//。。。
    	//调用父类的dispatchTouchEvent(也就是这个ViewGroup当做普通View处理，自己处理事件)
		return super.dispatchTouchEvent(ev);
	}
	
	
	//能走到这里，根据前面那三条，不是DOWN，而且target不为空，也就是DOWN已经被孩子处理
	if (!disallowIntercept && onInterceptTouchEvent(ev)) {//要拦截了
	//这个if一般是这样的：当子View上正在触发非DOWN(比如MOVE)事件，然后把该View，也就是这个
	//子View的父亲的disallowIntercept改变了，就走到这里了，父亲就开始拦截，然后就像下面那样，
	//给孩子分发一个CANCEL的事件，然后把mMotionTarget置为空，并且return true，当下个事件来到
	//的时候，在上面那一步，target已经等于null了，这样这个ViewGroup就会自己去处理事件了，然后就
	//走到onTouchEvent了，所以在这里if里面不用考虑onTouchEvent
		ev.setAction(MotionEvent.ACTION_CANCEL);
    	if (!target.dispatchTouchEvent(ev)) {
        	//这里是空的
            //这里主要作用是：我要拦截了，那么我就给我的孩子分发一个事件，ACTION是ACTION_CANCEL
		}
		//拦截了，以后不会让孩子去处理了，把mMotionTarget清空 
		mMotionTarget = null;
		return true;
	}
	
	
	if (isUpOrCancel) {
		mMotionTarget = null;
	}
	
	//把事件分发给孩子(如果return false，就会调用它父亲的这一行，一直往上，都是return false，
	//最后就到了Activity了,下一段代码分析)
	return target.dispatchTouchEvent(ev);	
}
```



### 前面说了，如果dispatchTouchEvent返回了false，会一直往上return，直到被Activity消费，上代码

```
Activity:
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    //window的superDispatchTouchEvent 如果return false，就往下走，调用了activity的onTouchEvent
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}

PhoneWindow：
public boolean superDispatchTouchEvent(MotionEvent event) {
	//调了mDecor.superDispatchTouchEvent(event)
	//其实就是ViewGroup的dispatchTouchEvent，回归到上一个代码分析了
    return mDecor.superDispatchTouchEvent(event);
}

综上，如果dispatchTouchEvent返回false，最终就会走到activity的onTouchEvent中
```





接下来分析View的dispatchTouchEvent

### View：

```
//看着很简单
public boolean dispatchTouchEvent(MotionEvent event) {
    if (!onFilterTouchEventForSecurity(event)) {
        return false;
    }
    //有了onTouchListener,就执行onTouch，没有，或者return false，才会走onTouchEvent
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
            mOnTouchListener.onTouch(this, event)) {
        return true;
    }
    //其实onClick是在onTouchEvent中的，所以onTouch->onTouchEvent->onClick/onLongClick
    return onTouchEvent(event);
}
```



> 都说事件分发机制搞清楚三个方法就ok了

- dispatchTouchEvent
  - view 搞定
  - viewGroup 搞定

- onInterceptTouchEvent 
  - view 没有
  - viewGroup 在dispatchTouchEvent的分析中说了

- onTouchEvent

  接下来开始分析




### onTouchEvent：

```
//这个类只存在于View中，用来处理事件(ViewGroup一般会把事件分发给孩子，让孩子来处理，如果自己处理
//就通过super class来处理，也就是View，所以还是它)
public boolean onTouchEvent(MotionEvent event) {
    final int viewFlags = mViewFlags;

	//如果view是DISABLED，那么这个View不可以响应事件，即不可以对事件作出回应，但是他是会
	//消耗事件的：1.CLICKABLE 2.LONG_CLICKABLE，这里直接根据这两个条件return，不论true or false
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }

	//委托给另一个View去处理？没用过
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

	//对事件处理，其实就是click和longclick(当然我们可以重写这个方法,但是对于view，系统
	//只帮忙处理这两个，scrollview等都是重写的)
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
            	//手指抬起，各种校验，最后决定调用点击、长按方法
                break;

            case MotionEvent.ACTION_DOWN:
            	//按下
                if (mPendingCheckForTap == null) {
                    mPendingCheckForTap = new CheckForTap();
                }
                mPrivateFlags |= PREPRESSED;
                mHasPerformedLongPress = false;
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                break;

            case MotionEvent.ACTION_CANCEL:
            	//事件取消，就会remveXXCallBack(),还会
                //refreshDrawableState();
                mPrivateFlags &= ~PRESSED;
                refreshDrawableState();
                removeTapCallback();
                break;

            case MotionEvent.ACTION_MOVE:
            	//手指移出去（View的可点击范围），就会remveXXCallBack(),还会
            	//refreshDrawableState();
            	//...
                if ((x < 0 - slop) || (x >= getWidth() + slop) ||
                        (y < 0 - slop) || (y >= getHeight() + slop)) {
                    // Outside button
                    removeTapCallback();
                    if ((mPrivateFlags & PRESSED) != 0) {
                        // Remove any future long press/tap checks
                        removeLongPressCallback();
                        // Need to switch from pressed to not pressed
                        refreshDrawableState();
                    }
                }
                break;
        }
        return true;
    }
	//不是DISABLE,没有被委托的View处理，且不可点击（click 和 longclick），直接return false
    return false;
}
```

