---
title: AndroidStudio查看快速跳转Hide代码
date: 2017-03-02 23:22:16
categories: 开发小技巧

---
> android源码中被@hide注解的类/方法是不可以在studio中通过类搜索,ctrl+鼠标左键,类的继承结构等方法看到的。

我们可以通过替换sdk目录/platforms/android-xx下的android.jar的方式来找出所有被隐藏的类，但是这样的话就可以直接使用这些类了，既然谷歌把它隐藏了，那随意使用肯定是有风险的，同时，如果不用反射，直接使用，部署在手机上也是会出问题的。所以可以替换某一个版本的jar,然后新建一个module,依赖那个完整的jar的版本,其他module依赖正常的版本,这样就既能轻松的查看完整源码，又能避免随意的使用hide的类了。