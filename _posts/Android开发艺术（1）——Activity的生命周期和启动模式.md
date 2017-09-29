---
title: Android开发艺术（1）——Activity的生命周期和启动模式
date: 2017-09-06 20:26:22
tags:
categories: 阅读笔记
---



# Activity生命周期的分析

从两种情况来分析

- 典型情况（用户正常参与的情况）
- 异常情况（由于内存不足，被系统杀掉、Configuration改变等）

## 典型情况下的生命周期分析

- onPause不能太耗时，因为上一个Activity执行完之后 下个Activity的onCreate/onStart/onResume/onResume才能执行，所以不要在onPause中执行耗时才做，使新的Activity尽快切到前台

  ```
  从Activity1到Activity2
  09-06 20:56:55.190 com.xk.chapter1 D/Activity1: onCreate-->
  09-06 20:56:55.210 com.xk.chapter1 D/Activity1: onStart-->
  09-06 20:56:55.210 com.xk.chapter1 D/Activity1: onResume-->
  09-06 20:56:59.500 com.xk.chapter1 D/Activity1: onPause耗时操作开始-->
  09-06 20:57:01.510 com.xk.chapter1 D/Activity1: onPause耗时操作结束-->
  												Activity2出现
  09-06 20:57:01.520 com.xk.chapter1 D/Activity2: onCreate-->
  09-06 20:57:01.520 com.xk.chapter1 D/Activity2: onStart-->
  09-06 20:57:01.520 com.xk.chapter1 D/Activity2: onResume-->
  09-06 20:57:01.990 com.xk.chapter1 D/Activity1: onStop-->

  从Activity2返回Activity1
  09-06 21:01:13.490 com.xk.chapter1 D/Activity2: onPause耗时操作开始-->
  09-06 21:01:15.490 com.xk.chapter1 D/Activity2: onPause耗时操作结束-->
  												Activity1
  09-06 21:01:15.490 com.xk.chapter1 D/Activity1: onRestart-->
  09-06 21:01:15.490 com.xk.chapter1 D/Activity1: onStart-->
  09-06 21:01:15.490 com.xk.chapter1 D/Activity1: onResume-->
  09-06 21:01:15.890 com.xk.chapter1 D/Activity2: onStop-->
  09-06 21:01:15.890 com.xk.chapter1 D/Activity2: onDestroy-->
  ```

- onStop可以做一些稍微重量级的操作，不过也别太耗时

- onDestory中可以做资源的释放等操作

- 弹出一个新的Activity，旧的会调onPause->onStop，特殊的，如果新的Activity是透明主题，那么旧的Activity不会调用onStop，因为它仅仅是失去了焦点，无法交互，还是可见的

- onStart、onStop是从是否可见角度来说的，onPause和onResume是从是否位于前台（可交互）角度来说的，除此之外没啥区别。注意第一条。。。

- 启动Activity的请求由Instrumentation来处理，它通过Binder向AMS发送请求，AMS内部维护着一个ActivityStack并且负责栈内Activity的状态同步，AMS通过ActivityThread去同步Activity的状态，从而完成生命周期的回调

## 异常情况下的生命周期

### 资源相关的系统配置发生改变导致Activity被杀死并重新创建

比如横屏和竖屏会使用不同的资源文件，如果屏幕方向发生改变，activity就会被销毁重建，除非我们指定不要让他销毁重建。当然，这种销毁重建会调用onSaveInstanceState来保存activity的状态，重建的时候会调用onRestoreInstanceState来恢复（onSaveInstanceState保存的数据，在onCreate和onRestoreInstanceState中可以拿到）

onRestoreInstanceState会在onStart之前调用

activity被意外终止的时候，activity会调用onSaveInstanceState，然后委托window，接着window委托他内部的view，也就是顶层View（一般是DecorView），然后DecorView委托子View。。。一直往下到每一个View。每个View都有onSaveInstanceState和onRestoreInstanceState方法的。这种机制类似于***事件分发机制***

onSaveInstanceState只有在activity即将被被销毁，并且有机会重建的时候才会被调用。简单理解为就是异常终止，并且会马上重建，对比按返回键销毁和旋转屏幕销毁就好理解了。actvity销毁后保存的数据在onCreate和onRestoreInstanceState中都可以拿到，但是onCreate中需要做非空判断，onRestoreInstanceState不需要，因为onRestoreInstanceState一调用，一定是有数据需要恢复了

按Home键或者启动新Activity仍然会单独触发onSaveInstanceState的调用。因为Home之后或者新的activity出现之后，旧的activity到了后台，都有可能被杀。

### 资源内存不足导致低优先级Activity被杀

activity的优先级

- 前台Activity
- 可见但非前台，比如弹出dialog，我的理解是onPause之后的，不过不一定对，有待探究
- 后台Activity，我的理解是onStop之后的，任玉刚书中说的是暂停后的，感觉有歧义，不太确定，有待探究

如果一个进程中没有四大组件在执行，很快会被杀，所以后台任务一般要在service中运行。我的理解是，如果在activity中开一个线程去做后台任务，activity不在前台的时候就容易被杀，比如回到桌面

如果不想在屏幕旋转后重建activity可以在清单文件中指定configChanges属性，orientation、keyboardHidden，API13之后还需要指定screenSize。去年学这块就是死记，其实现在看来，所有的config（具体查表）改变都会导致Activity重建，这里配置之后，被配置的选项改变不会使activity重建，但会回调onConfigurationChange方法，通知我们改变了，然后根据自己的需求做一些事情就好了。

# Activity的启动模式

## Activity的LaunchMode

当一个任务栈中没有activity的时候，这个栈就会被回收

几种启动模式：

- standard

- singleTop

  栈顶复用，当被复用的时候，会调用onNewIntent，参数中可以取到启动新activity携带的intent，并且onCreate、onStart不会被调用

- singleTask

  被启动的activity查看是否存在他想要的任务栈，如果存在，就是那样了。。。不存在，就创建任务栈（默认跟启动他的context一样，可以通过taskaffinity指定）

- singleInstance

> adb shell dumpsys activity 查看栈信息

启动一个activity的时候，一般新的activity会在调用startactivity的context所在的任务栈中，但是applicationcontext不存在于任务栈，所以需要为新的activity指定FLAG_ACTIVITY_NEW_TASK标记，不明白为什么在清单文件中指定不行

**TaskAffinity**（任务相关性）可以指定一个activity所需的栈名（默认为他的包名），他只能和singleTask和allowTaskRepareniting配合使用，其他情况无效

如果没有指定TaskAffinity，新启动的activity会继承启动他activity的栈(singleInstance会进入新的栈，但是实验发现，栈名是一样的，不过不是一个对象，相当于创建一个新的栈名相同的栈)

allowTaskReparenting可以指明一个activity

onNewIntent被调用的那些情况下，onCreate、onStart不会被调用

### Activity的flags

>摘自Android开发艺术图书勘误：第1.2.2小节中，Activity的Flags，这一节的内容直接翻译了Android官方文档（<http://developer.android.com/guide/components/tasks-and-back-stack.html#TaskLaunchModes>），但是经过实例验证，发现书中的描述不准确（或者说官方文档中的描述不准确），结论为：Flags并不能简单地等同于启动模式，这一块内容需要进一步验证。

# IntentFilter的匹配规则

这个直接结合生活就很好理解了

action/data在intent中是set，category是add，所以就知道前两个是唯一的，category是可以配多个的

- action行为、动作

  - 在清单文件中配置的时候，action表示“我可以干什么”，所以可以配置多个：我可以展示图片、我可以展示音频
  - 在代码中指定意图的时候只能指定一个action：我将要干什么，比如我要展示图片，或者我要播放音频，则分别跳转到相应的activity，不能我又要播放音频又要展示图片
  - 所以action就是清单文件中配置多个，指定意图的时候只能制定一个

- category分类
    - 在清单文件中配置的时候，配置我属于什么什么，比如：我属于默认分类、运动类、户外类、轮滑类等（注意，这里最好指明自己属于默认类，这样在查找的时候，别人都能找到）
    - 在代码中指定意图的时候可以指定我要：运动类并且是户外类，或者轮滑类（注意，系统会自动加上默认类）
    - 所以，代码中指明的类别，一定要在清单文件中指明，比如一个activity属于默认分类、运动类、户外类、轮滑类，在代码中我可以直接找运动或者运动以及户外，但是如果我要找游戏，肯定是找不到了
  - data数据形式
      - 这个跟action类似，他是根据数据的规则来匹配的，action是我要找到"可以干什么"的activity，data是我要找到“可以接收该data”的activity
      - data主要分为URI和mimeType两部分，一个代表资源位置，一个代表资源类型

对于Service和BroadcastReceiver，这套匹配规则也适用，不过对于Service，官方推荐使用显示意图

在使用隐式意图的时候，最好先做个判断，判断是否有可以匹配的activity，具体方法查就行了；在清单文件中配置category的时候，最好配置上default，使得intent更容易匹配它，因为每个intent都会默认加上default