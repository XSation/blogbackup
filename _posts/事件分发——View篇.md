---
title: 事件分发——View篇
date: 2016-11-25 10:34
categories: view
---
[郭神的博客](http://blog.csdn.net/guolin_blog/article/details/9097463/)

代码都是只考虑了最简单的情况，最简单的情况有利于搞懂机制

我用的是android 24的源码，与郭神的有一点出入，大概看看就好


**单独分析View,不考虑ViewGroup**




## 1.一个view的事件产生之后会首先调用dispatchTouchEvent方法，决定是否分发该事件


**View#dispatchTouchEvent方法：**
```
//返回true表示该View 处理这个事件 否则false
 public boolean dispatchTouchEvent(MotionEvent event) {
      
        boolean result = false;

        
        if (onFilterTouchEventForSecurity(event)) {
         

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {//如果注册了onTouch事件,前两个条件满足，第三个条件表示是否为enable，第四个条件会执行onTouch方法，它也是最后一个条件，他的返回值决定了该view是否处理事件（由于一个事件产生后会先调用dispatchTouchEvent，然后我们分析得onTouch方法优先级挺高的，毕竟现在还没有看到onClick方法，这里的onTouch就是注册touch事件的时候我们实现的那个方法）
                result = true;
            }
//如果上面的条件没有满足，那么就会执行View#onTouchEvent方法
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }


        return result;
    }
```

## 2.如果上面的dispatchTouchEvent在onTouch中就返回了true，那么就结束了，如果没有，就会继续向下执行如上代码，执行View#onTouchEvent(猜想onClick那些会在这里)

**View#onTouchEvent**

```
//返回true表示处理的事件，否则return false。 同时它return false则表示dispatchTouchEvent返回了false，它返回true表示dispatchTouchEvent返回了true
 public boolean onTouchEvent(MotionEvent event) {
      
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {//view有click或者longClick，进入下面代码，否则return false（全部源码还有一些别的判断，这里不考虑）
            switch (action) {
                case MotionEvent.ACTION_UP:
                   performClick();//抬起手指，执行点击（该方法就是判断是否实现了onClick事件，然后调用它）
                    break;

                case MotionEvent.ACTION_DOWN:
                    
                    break;

                case MotionEvent.ACTION_CANCEL:
                    
                    break;

                case MotionEvent.ACTION_MOVE:
                    
                    break;
            }

            return true;
        }

        return false;
    }
```
onTouchEvent里面就是如果有点击事件，并且可以点击（setOnClickListener（）方法里面设置了clickable为true），当action为up的时候，就执行点击。并且只要有点击事件，就返回true，否则返回false。
**注意**：down->move->...->move->up  这是一个事件序列，只要有一个返回了false，后面的就不会被接收到。 但是尽管我们在onTouch中返回了false，但是会在onTouchEvent中如果有点击事件，会进入到switch那段代码，最后返回的还是false。所以button注册了ontouch事件后，一直可以收到action，因为他有clickable，但是imageview默认不可点击，所以在onTouchEvent中不会进入switch那块，最后返回了fasle;
换种说话就是，如果onTouch中返回了true，表示该view要处理这一串事件，所以他可以一直接受到，如果返回了false，表示在onTouch中view没说要处理，但是如果他是可点击的，在接下来的onTouchEvent中，view表明要处理这一串序列了，所以依旧可以接收到，但是imageview是不可点击的，所以他依旧返回false，这样dispatchTouchEvent就返回了false，所以事件就会开始分发。




> copy一下郭神的问答：
>  **onTouch和onTouchEvent有什么区别，又该如何使用？**
> 从源码中可以看出，这两个方法都是在View的dispatchTouchEvent中调用的，onTouch优先于onTouchEvent执行。如果在onTouch方法中通过返回true将事件消费掉，onTouchEvent将不会再执行。
> 另外需要注意的是，onTouch能够得到执行需要两个前提条件，第一mOnTouchListener的值不能为空，第二当前点击的控件必须是enable的。因此如果你有一个控件是非enable的，那么给它注册onTouch事件将永远得不到执行。对于这一类控件，如果我们想要监听它的touch事件，就必须通过在该控件中重写onTouchEvent方法来实现。


