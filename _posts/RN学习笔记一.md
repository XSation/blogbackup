---
title: RN学习笔记一
date: 2017-07-21 12:31:27
tags:
---

```
//source，style后面跟的花括号表示里面应该放一个js变量或者表达式
<Image source={pic} style={{width:193,height:100}}/>
```

> 可以使用props和state两种数据来控制控件

- props：在父控件中设置，一经设置，在控件的生命周期中不能被改变
- state：对于需要改变的情况，可以使用state


```
const {a,b}=c//表示c是一个对象，取C中的名为a、b的参数，赋值给a，b
```



