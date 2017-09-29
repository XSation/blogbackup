---
title: TV开发下拉加载更多遇到的坑
date: 2017-07-14 15:05:36
categories: 踩过的坑
tags:
---

recyclerview上拉加载更多在手机上很容易实现，网上的demo也有很多，盒子开发也类似：焦点所在的条目是最后一**行**的时候，按下，加载更多即可，为了保证焦点不丢失（手机中不用担心这一点，盒子上焦点item一般都有加特效），不可以更新全部item，所以只能使用notifyItemRangeInserted这个方法了，看着很简单。**那么坑来了**

由于当前item贴最下边，加载出新的item之后，下一行的并没有露出了，如果直接再按下，是没法下移的，只要按了左右，recyclerview会自动调整位置，使下一行露出一点，然后才可移动。

### 解决方案

```java
searchAdapter.notifyItemRangeInserted(startIndex,data.size());
tvRecyclerView.post(()->tvRecyclerView.scrollBy(0,1));
```

靠，如此简单，坑死哥了