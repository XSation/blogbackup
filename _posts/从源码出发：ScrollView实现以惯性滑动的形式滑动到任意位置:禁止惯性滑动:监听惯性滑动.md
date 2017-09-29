---
title: 从源码出发：ScrollView实现以惯性滑动的形式滑动到任意位置/禁止惯性滑动/监听惯性滑动
date: 2016-11-04 21:55
categories: view
---
**读完这篇博客可以实现：**
	1.scrollview从任意位置通过惯性滑动到任意位置
	2.获取手离开屏幕后惯性滑动的距离（时间也可以）
	3.既然可以控制惯性滑动了，那么有时候惯性滑动造成的各种被重复触发事件导致的bug也就可以解决了。


**先上三个干货**

-  在scrollview中重写fling（）方法，他的参数是用户滑动时，松开手的那一瞬间的初速度，他决定了view惯性滑动的时间和距离，这个参数可以根据需求传入任意值，从而达到各种效果。

-  传入你要到达的位置（scrollY），计算出需要的初速度（这个初速度可以在上面那条中用）

```
/**
     * 通过目标y得到需要的初速度 （指头向上滑，需要的初速度）
     * @param endY
     * @return
     * @author xuekai
     */
    public int getVelocityY(int endY){
        int signum=-1;//注意：如果指头上滑，他肯定是-1，如果下滑，把这里改成1，我的需求肯定是上滑，所以写死了
        double dis=(endY-upY)*signum;
       double g= Math.log(dis/ViewConfiguration.getScrollFriction()/ (SensorManager.GRAVITY_EARTH // g (m/s^2)
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f))* Math.log(0.9)*((float) (Math.log(0.78) / Math.log(0.9)) - 1.0)/(Math.log(0.78) );
       return (int) (Math.exp(g)/0.35f*(ViewConfiguration.getScrollFriction() * SensorManager.GRAVITY_EARTH
                       * 39.37f // inch/meter
                       * (context.getResources().getDisplayMetrics().density * 160.0f)
                       * 0.84f));
    }
```

- 通过上面第一条中的初速度可以计算出它将会滑动的距离（scrollY）

```
/**
     * 
     * @param velocityY 初速度
     * @return 滑动的距离
     * @author xuekai
     */
 public double getDis(int velocityY) {

        final double l = getG(velocityY);
        final double decelMinusOne = (float) (Math.log(0.78) / Math.log(0.9)) - 1.0;
        return ViewConfiguration.getScrollFriction() * (SensorManager.GRAVITY_EARTH // g (m/s^2)
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f) * Math.exp((float) (Math.log(0.78) / Math.log(0.9)) / decelMinusOne * l);
    }


    public double getG(int velocityY) {
        return Math.log(0.35f * Math.abs(velocityY) / (ViewConfiguration.getScrollFriction() * SensorManager.GRAVITY_EARTH
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f));

    }
```

**开始扯（走）犊（源）子（码）**
		我准备利用scrollview实现一个下拉刷新的功能，scrollview初始状态下让他的向上滚动1000（假设值）。然后在onTouch事件中做处理：只要是触发action_up，并且scrollY小于1000，就自动滚动到1000，同时，在onScrollChanged中也做一个判断，当处于抬起状态（在action_down和action_up中设置boolean）并且scrollY小于1000，也滚动到1000.这样一个简单地下拉刷新实现了。
	可是scrollTo这个方法会让它瞬间到了某个位置，没有下拉刷新中缓慢上升的效果，怎么办呢？ 好说，值动画嘛！！！
	巴拉巴拉巴拉，写完了，但是发现没效果，原因是松开手之后触发了action_up，立即弹回去了。去掉它，只用onScrollChanged，又会受到惯性滑动的影响（惯性滑动的时候，已经是抬起状态了。）
	怎么办，于是开始百度（很少用谷歌，哈哈），百度出来的都没用。于是开始问人，问不到。还是自己看源码吧。
	发现ScrollView中有一个方法 fling（int velocityY）


```
 /**
     * Fling the scroll view
     *
     * @param velocityY The initial velocity in the Y direction. Positive
     *                  numbers mean that the finger/cursor is moving down the screen,
     *                  which means we want to scroll towards the top.
     */
    public void fling(int velocityY) {
        if (getChildCount() > 0) {
            int height = getHeight() - mPaddingBottom - mPaddingTop;
            int bottom = getChildAt(0).getHeight();

            mScroller.fling(mScrollX, mScrollY, 0, velocityY, 0, 0, 0,
                    Math.max(0, bottom - height), 0, height/2);

            if (mFlingStrictSpan == null) {
                mFlingStrictSpan = StrictMode.enterCriticalSpan("ScrollView-fling");
            }

            postInvalidateOnAnimation();
        }
    }
```
用渣的一比的英语看完了注释，这个参数是松手那一瞬间的加速度。由于我的需求中，手指上滑不会出现这样的问题，下滑才会，于是产生了这样的代码（在自定义scrollview中重写它）

```
  @Override
    public void fling(int velocityY) {
        if (velocityY < 0) {
            super.fling(0);
            return;

        } else {
            super.fling(velocityY);
        }
    }
```
这样的话问题解决了，并且在上滑的时候也有惯性滑动，但是上滑就变得死气沉沉，因为上滑的初速度都是0了，这咋行，继续看。

在上面那个方法里调用了 	
```
mScroller.fling(mScrollX, mScrollY, 0, velocityY, 0, 0, 0,
                    Math.max(0, bottom - height), 0, height/2);  
```
点进去看		

```
public void fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY, int overX, int overY) {
        // Continue a scroll or fling in progress
        if (mFlywheel && !isFinished()) {
            float oldVelocityX = mScrollerX.mCurrVelocity;
            float oldVelocityY = mScrollerY.mCurrVelocity;
            if (Math.signum(velocityX) == Math.signum(Math.signum(velocityY) == Math.signum(oldVelocityY)) {
                velocityX += oldVelocityX;
                velocityY += oldVelocityY;
            }
        }
        mMode = FLING_MODE;
        mScrollerX.fling(startX, velocityX, minX, maxX, overX);
        mScrollerY.fling(startY, velocityY, minY, maxY, overY);
    }
```


发现有一行代码是正对Y方向的fling的（猜，看源码就是要猜，哈哈）
跟着继续走	

```
getSplineFlingDuration(velocity);
getSplineFlingDistance(velocity);
```
发现了这么两个方法。用英语3.9级的水平来看，我草，太开心了，这岂不是飞的距离，飞的时间都有了吗，那这样我就可以控制“飞行”了。
当然，肯定不是那么好调用的，不然不用费那么大劲了，上反射。
巴拉巴拉巴拉，我操，最后一步竟然没成功，静态内部类。
怎么办？调用不到照着写一个吧。。。
巴拉巴拉巴拉。。。


写好了，测试了一下，求出来的不是fling的时间，沮丧。
妈的，继续撸。。。
看了注释才发现，这个是fling的减速度。
那还好，辛苦没有白费，继续干。。。
终于找到了求惯性滑行距离的方法了，调用不到，只能仿照出来	
```
 public double getSplineFlingDistance(int velocityY) {

        final double l = getG(velocityY);
        final double decelMinusOne = (float) (Math.log(0.78) / Math.log(0.9)) - 1.0;
        return ViewConfiguration.getScrollFriction() * (SensorManager.GRAVITY_EARTH // g (m/s^2)
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f) * Math.exp((float) (Math.log(0.78) / Math.log(0.9)) / decelMinusOne * l);
    }


    public double getG(int velocityY) {
        return Math.log(0.35f * Math.abs(velocityY) / (ViewConfiguration.getScrollFriction() * SensorManager.GRAVITY_EARTH
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f));

    }
```
这样就能得到了，也就是你在每次松开滑动的手的时候，可以知道他将滑动到的位置。可是这有什么卵用（不过有的一些需求下有用，对于我没用）。 正好反了，我想要通过我要滑动到的位置来计算需要的初速度。

于是开始尝试逆着算，数学里这叫还原法，不过不一定能还原出来。经过一系列的百度数学函数意思，结合高中忘得差不多的数学水平，终于还原出来了。

```
 /**
     * 通过目标y得到需要的初速度 （指头向上滑，需要的初速度）
     * @param endY
     * @return
     */
    public int getVelocityY(int endY){
        int signum=-1;//如果指头上滑，他肯定是-1；
        double dis=(endY-upY)*signum;
       double g= Math.log(dis/ViewConfiguration.getScrollFriction()/ (SensorManager.GRAVITY_EARTH // g (m/s^2)
                * 39.37f // inch/meter
                * (context.getResources().getDisplayMetrics().density * 160.0f)
                * 0.84f))* Math.log(0.9)*((float) (Math.log(0.78) / Math.log(0.9)) - 1.0)/(Math.log(0.78) );
       return (int) (Math.exp(g)/0.35f*(ViewConfiguration.getScrollFriction() * SensorManager.GRAVITY_EARTH
                       * 39.37f // inch/meter
                       * (context.getResources().getDisplayMetrics().density * 160.0f)
                       * 0.84f));
    }
```

墨迹完了。。。