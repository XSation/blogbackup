---
title: App开发者需要掌握的底层知识（1）
date: 2017-07-12 23:36:28
tags:
categories: App开发者需要掌握的底层知识
---

App开发者需要掌握的底层知识:
-  Binder

-  AIDL

-  AMS

-  Activity

-  Service

-  ContentProvider

-  匿名共享内存

-  BroadcastReceiver

-  PMS及App安装过程


  以上为该系列所包括的，除此之外还包括:

- View和ViewGroup

- Message、Looper和Handler

- 权限管理

- Android SDK工具内部原理

以上即可



# Binder

对于binder，不需要了解具体的实现，它涉及到了C++层面

binder是为了解决进程间通讯

1. Binder可以分为client和server两个进程，可以粗略的认为，发消息的一端就是binder client，接受消息的一端就是binder server。AIDL、四大组件和AMS之间的通讯，其实就是binder两个进程之间的通讯。


# AIDL与AMS相关

一个aidl文件

```
interface IComputer {
    int add(int a, long b);
}
```

make项目之后，会自动生成一个java文件

```
interface IComputer extends IInterface{
  
  public abstract class Stub extends Binder implements IComputer{
    
    public IComputer asInterface(IBinder ibinder){
      //Stub表示对于IComputer具体的实现
      //如果stub的定义与使用在同一进程，则直接返回this
      //如果不在同一进程，new一个代理类，并返回
    }
    
    public class Proxy implements IComputer{
      
    }
  }
  
}
```

在服务的定义IComputer的具体实现

```
public class Computer extends IComputer.Stub{

        @Override
        public int add(int a, long b) throws RemoteException {
            return 0;
        }
    }
```

在客户端，或者服务端使用

```
IComputer.Stub.asInterface(iBinder).add(1,2);
```

asInterface这个方法被调动的时候，如果是在本地，从本地取出IComputer，如果不在一个进程中，则创建一个proxy，并且返回，然后执行的方法就是proxy的方法。

---

类比上面：

IActivityManager => IComputer

ActivityManagerNative => IComputer.Stub

ActivityManagerProxy => Proxy

> ActivityManagerNative和ActivityManagerProxy都是针对使用者这一端的，ActivityManagerServer是在服务端的，它继承自ActivityManagerNative

```
ActivityManagerNative.asInterface().startActivity();
```

如果asInterface返回的是AMN，说明服务端和客户端是在同一个进程中的，如果返回的是AMP，说明不是在同一个进程中，AMP中的方法（startActivity）是把数据写入到另一个进程（AMS那边）,然后ams那边计算出结果，然后写入到amp这端

