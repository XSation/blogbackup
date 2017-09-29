---
title: 仿《开眼App》轮播图动画
date: 2017-08-23 08:50:25
tags:
categories: view
---



写了一晚上，写到电脑没电，没实现，总是有点小问题，睡觉前想了想，果然思路清晰，早上来公司几分钟搞定

先上效果图

![](/pic/lunbotu.gif)

给viewpager设置PageTransformer

***真的炒鸡简单呀，日了狗了***

```kotlin

/**
 * Created by xuekai on 2017/8/22.
 */
class HomeBannerTransformer : ViewPager.PageTransformer {
    override fun transformPage(page: View?, position: Float) {
        val width: Int = page?.width!!
        //以向左滑动为例
       // if (position <= 0) {//中间的
            page?.scrollX = (position * width).toInt() / 4 * 3
        //} else if (position <= 1) {//右侧的
           //page?.scrollX = (position * width).toInt() / 4 * 3

        //}
    }

}
```

