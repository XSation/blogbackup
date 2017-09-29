---
title: Android调用系统相机拍照 返回数据为空
date: 2016-12-13 10:52
categories: 踩过的坑 

---
调用相机拍摄后，会返回到上一个activity，调用onActivityResult。

- 如果在跳转的时候，指定了保存uri，系统就会把原图保存到那个uri，在onActivityResult中通过uri获取bitmap，但是不知道为啥，会有延时，不是及时可以获取到，所以可以写一个task，用whlie循环，获取bitmap，直到获取到为止（由于设置了uri，所以在data的data这个key中拿不到数据，为空）
- 如果没有指定uri，系统会保存在默认路径，并且在onActivityResult的data中返回一个bitmap，通过“data”这个key可以获取到，但是是缩略图，不清晰，这个是可以及时拿到的。
