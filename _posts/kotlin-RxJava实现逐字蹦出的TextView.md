---
title: kotlin+RxJava实现逐字蹦出的TextView
date: 2017-08-22 20:43:58
tags:
categories: view
---

> 最近在学习kotlin，所以决定用kotlin写一个仿"开眼"App，现在github上有一个写好的，clone下来看了之后发现很多漂亮的效果没有实现，所以决定自己写一个。



#### 项目中有一个TextView比较酷炫，也不知道网上有没有，直接撸一个

效果如图：

![](/pic/jumpshowtextview.gif)



```kotlin
package com.xk.eyepetizer.ui.view

import android.content.Context
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.graphics.Rect
import android.util.AttributeSet
import android.widget.FrameLayout
import android.widget.LinearLayout
import com.xk.eyepetizer.io_main
import io.reactivex.Observable
import io.reactivex.disposables.Disposable
import java.util.concurrent.TimeUnit

/**
 * Created by xuekai on 2017/8/22.
 */
class JumpShowTextView : FrameLayout {
    constructor(context: Context?) : this(context, null)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs) {
        init()
    }


    var paint: Paint? = null
    var content: String? = ""

    var isBold: Boolean = false
    var color: Int = Color.BLACK

    var textSize: Float = 52F

    //线程正在运行
    var isRun: Boolean = false

    var marginBottom: Float = 0f

    var text: String? = ""
        set(value) {
            if (isRun) {

            }
            field = value
            computeViewSize()
            start()
        }


    private fun computeViewSize() {
        var paint = Paint()
        paint.setTextSize(textSize);
        var rect = Rect()   // 使用上面的画笔最终绘制出字符串所占的矩形
        paint.getTextBounds(text, 0, text!!.length, rect); // 四个参数分别为字符串，起始位置，结束位置，矩形
        val textWidth = rect.width()
        val textHeight = rect.height()
        val layoutParams = LinearLayout.LayoutParams(textWidth, textHeight)
        layoutParams.bottomMargin = marginBottom.toInt()
        this.layoutParams = layoutParams

        invalidate()
    }


    private fun init() {
        paint = Paint()
        setBackgroundColor(0x00000000)
    }

    var subscribe: Disposable? = null

    fun start() {
        if (isRun) {
            subscribe?.dispose()
        }
        content = ""

        paint?.isFakeBoldText = isBold
        paint?.color = color
        paint?.setTextSize(textSize);

        isRun = true
      //这里用rxjava实现，比直接用普通的方式实现简单了太多，之前是通过普通的方式实现，由于我这个start方法会调用多次，且每次调用后text不同，且长度不同，导致这块经常出现数组越界，控制线程也不太方便
        subscribe = Observable.interval(80, TimeUnit.MILLISECONDS)
                .take(text?.length!!.toLong())
                .io_main()
                .subscribe({ i ->
                    content = content + text!![i.toInt()]
                    invalidate()
                }, { e -> e.printStackTrace() }, { isRun = false })

    }


    override fun onDraw(canvas: Canvas?) {
        super.onDraw(canvas)
        val fontMetrics = paint?.getFontMetrics()

      
      //这块写法有很多种，主要就是想办法确定了文字的垂直位置即可，这里列出了一个公式，可以记录下来，以后就不用计算了，直接拿来用
        val baseLine1 = measuredHeight / 2 - (fontMetrics!!.top + fontMetrics?.bottom) / 2
//drawText的第二个参数的值=要让文字的中心放在哪-（fontMetrics.top+fontMetrics.bottom）/2
//此时求出来的baseline可以使文字竖直居中

        canvas?.drawText(content, 0f, baseLine1, paint)


    }
}
```

