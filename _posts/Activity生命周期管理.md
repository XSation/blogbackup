---
title: Activity生命周期管理
categories: DroidPlugin
date: 2017-04-19 
     
---

> 参考文章[Android 插件化原理解析——Activity生命周期管理](http://weishu.me/2016/03/21/understand-plugin-framework-activity-management/)

java中实现动态加载,直接用`ClassLoader`,android中也是可以的,但是直接把某个类加载进来是不行的,activity、service等这些组件都是有生命周期的,都是由AMS统一管理的,所以仅仅直接加载进来是不行的,还需要交给AMS管理.

DroidPlugin通过`hook`机制最终赋予activity、service等组件生命周期

## AndroidManifest.xml的限制
在android中,要启动一个activity,必须要在清单文件中声明,那么就可以先在清单文件中声明有限个activity,要启动某个activity的时候,先启动声明中的,然后在某个时机替换为我们需要启动的activity

## Activity启动过程

未完待续。。。
