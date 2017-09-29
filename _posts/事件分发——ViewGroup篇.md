---
title: 事件分发——ViewGroup篇
date: 2016-11-27 13:02
categories: view
---
[郭神的博客](http://blog.csdn.net/sinyu890807/article/details/9153747)

代码都是只考虑了最简单的情况，最简单的情况有利于搞懂机制

```
public boolean dispatchTouchEvent(MotionEvent ev){

    boolean consume = false;
    if(onInterceptTouchEvent(ev)){
        consume = onTouchEvent(ev);
    }else{
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```

我用的是Android 24的源码，与郭神的有一点出入，大概看看就好

**结合ViewGroup分析。首先需要上篇的分析（事件分发学习笔记（View篇））**


## ****1.ViewGroup中有一个方法onInterceptTouchEvent，他默认返回false。返回结果表示是否拦截事件。


```
public boolean onInterceptTouchEvent(MotionEvent ev) {  
    return false;  
}  
```

## 2.ViewGroup是View的子类，所以他的dispatchTouchEvent方法也是会首先被调用的（如上一篇中分析的一样），当一个触摸事件发生的时候，首先会调用ViewGroup的这个方法。


```
public boolean dispatchTouchEvent(MotionEvent ev) {  
    //...
        if (disallowIntercept || !onInterceptTouchEvent(ev)) {//第一个默认是false，可以调用requestDisallowIntercept来改变它，但是我们不考虑，直接看第二个。也就是如果onInterceptTouchEvent返回了false，也就是不拦截touch事件，就会进入下面的内容。拦截了就不会进入。所以可以猜想，他的事件是从这里传递到他的孩子的。
            ev.setAction(MotionEvent.ACTION_DOWN);  
            final int scrolledXInt = (int) scrolledXFloat;  
            final int scrolledYInt = (int) scrolledYFloat;  
            final View[] children = mChildren;  
            final int count = mChildrenCount;  
            for (int i = count - 1; i >= 0; i--) {//从这里可以看出，他就是要遍历自己的孩子
                final View child = children[i];  
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE  
                        || child.getAnimation() != null) { 
                    child.getHitRect(frame);  
                    if (frame.contains(scrolledXInt, scrolledYInt)) { //这里是判断，这个孩子是否在touch发生的区域内（比如触摸一个地方，看这个孩子是否被触摸到） 
                        final float xc = scrolledXFloat - child.mLeft;  
                        final float yc = scrolledYFloat - child.mTop;  
                        ev.setLocation(xc, yc);  
                        child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
                        if (child.dispatchTouchEvent(ev))  {//在这里，就会调用孩子的dispatchTouchEvent方法，这就跟上一篇博客里讲的一样了。如果返回false，说明这个孩子不处理，继续循环下一个孩子，直到有一个孩子返回了true，表明他处理了事件，也就结束了，如果全部没有返回true，就继续往下走。  
                            mMotionTarget = child;  
                            return true;  
                        }  
                    }  
                }  
            }  
        }  
    }  
    boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||  
            (action == MotionEvent.ACTION_CANCEL);  
    if (isUpOrCancel) {  
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;  
    }  
    final View target = mMotionTarget;  
    if (target == null) {  //上面的孩子如果全部没有处理事件，那么ViewGroup的dispatchTouchEvent（该方法）就会执行到这里，target也是空的，所以就会进入这个判断，最后执行他父类的dispatchTouchEvent方法，也就是View#dispatchTouchEvent,就跟上一篇一样了，看看该控件是否会消费，如果消费，就返回true，然后返回给他的调用者，一级一级往回返，直到activity中，就结束了。不消费，那么dispatchTouchEvent返回false，给了他的调用者，也就是他的父容器，再看他的父容器的父类View是否会消费，直到activity。（举个列子：可滑动的一个容器，他是可以消费事件的，他包含了一个子控件。如果子控件可以消费事件会有两种情况：1.父容器拦截事件，那么子控件是获取不到的。2.如果父容器不拦截，那么子控件就消费了。   如果子控件不消费事件，还有两种情况：1.父容器拦截，那么就直接消费了。2.父容器不拦截，会询问子控件，子控件不消费，又回给父容器，父容器消费掉。  当然，如果父容器不消费，那就继续给父容器的父容器，直到给了activity。）
 
        ev.setLocation(xf, yf);  
        if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
            ev.setAction(MotionEvent.ACTION_CANCEL);  
            mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        }  
        return super.dispatchTouchEvent(ev);  
    }  
   //...
}  
```