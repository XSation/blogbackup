---
title: View的测量
date: 2016-11-28 21:33
categories: view
---
学安卓一年多了，关于view的绘制流程看过N遍，onLayout和onDraw比较简单，onMeasure一直没搞明白，大概是比较笨，理解能力太差，网上博客里的内容看了几次，也没太明白，今晚有仔细的研读了一遍任玉刚和鸿洋的博客，刚开始也没搞懂，越看越瞌睡。
突然看到鸿洋说的一句话，然后又想起来之前没明白的任玉刚的那个图，然后恍然大悟。
![view的celiang_p1](/pic/view的celiang_p1.png)
![view的celiang_p2](/pic/view的celiang_p2.png)


如图二所示，前面两行好理解，
第一行：一个view指定了具体的宽高，XXdp，那么无论他的父容器在他的爷爷容器中是如何的，该view都是显示XXdp。
第二行：如果一个view是match_parent,那么他的宽高就是他爸剩余容量的大小。
**第三行：**这里我之前理解错了，以为一个view设置了wrap_content之后，他的宽高就如右侧显示，当然，确实如此，但是总感觉有点疑问。看完鸿洋那句话才恍然大悟。其实这里我们也不用考虑他，因为view都说了他的宽高根据内容而定，所以他的父亲是给不了一个靠谱的答案的，所以还不如我们自己去计算。而现有的控件（TextView等）他们内部已经做了计算了。所以自定义view的时候，我们在onMeasure这块着重计算当宽高为wrap_content这种情况的宽高就好了，根据内容、边距等去计算，然后调用setMeasuredDimension即可。而对于前两种情况，一般用参数中传递过来的值（父亲给的）来设置就好了，因为前两种情况中父亲给的值是比较靠谱的。

附上参考链接：1.[任玉刚](http://blog.csdn.net/singwhatiwanna/article/details/38426471)

2.[鸿洋](http://blog.csdn.net/lmj623565791/article/details/38339817)