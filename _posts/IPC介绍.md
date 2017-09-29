---
title: IPC介绍
date: 2017-01-04 17:06
categories: IPC

---
# 1. android中的多进程模式

正常情况下，android中的多进程是指一个应用中的多个进程，所以这里不考虑两个应用的通讯。开启多进程的唯一方式就是在清单文件中给四大组件指定android：process属性。 
除此之外，还有一种非常规的方式，就是通过JNI在native层去fork一个新进程，暂不考虑。

```
   <activity android:name=".MainActivity"
            android:process="com.xk.otherprocess"/>
            <--这样写叫做完整的命名方式，产生的进程名就是这里指定的，不会附加包名信息，属于全局进程，其他应用可以通过ShareUID方法和它跑在同一个进程中 -->
```
```
   <activity android:name=".MainActivity"
            android:process=":remote"/>
            <--这样写是一种简写的方法，会在冒号前面添加包名，进程名为 包名:remote,属于当前进程的私有进程，其他应用组件不可以和它跑在同一进程中 -->
```

android会为每个应用分配唯一的UID，具有相同UID的应用才能共享数据。两个应用有相同的ShareUID并且签名相同，才可以跑在同一个进程，此时他们就可以相互访问私有数据——data目录，组件信息等。如果跑在一个进程，内存数据也可以共享了。


如果SecondActivity被指定在一个独立的进程中，那么MainActivity和SecondActivity同时访问一个类的static变量的时候，访问的不是一个东西，是两个副本。换句话说，运行在不同进程中的四大组件，只要他们需要通过内存来共享数据，那就会失败。	

- 静态成员和单例模式完全失效（由于无法共用内存）
- 线程同步机制完全失效（由于无法共用内存）
- SharedPreferences可靠性下降（sharedpreferences不支持多进程同时去执行写操作，并发写先让会出现问题，底层就是读写xml）
- Application会多次创建（一个组件跑在一个新的进程中，同时会分配独立的虚拟机，这个过程其实就是启动一个应用的过程）

综上所述：这样的多进程模式，其实就是为每一个进程分配了独立的虚拟机，Application和内存空间，会给我们带来很多困扰，也可以理解为：两个不同的应用采用了shareduid模式

# 2. IPC基础概念介绍

## Serializable
- static修饰的属于类，不属于对象，所以不参数序列化
- 用transient标记的也不参与序列化
- 使用简单，但是需要大量的io操作，开销大
- 关于serialVersionUID要说的几点
- 如果没有指定serialVersionUID，那么序列化或者反序列化的时候会通过计算类的结构（hash值），自动生成一个值，这样如果类结果发生了改变，那就反序列化失败了（因为反序列化的时候会先比较serialVersionUID是否相同）
- 如果手动指定，那么如果结构改变（增加或删除几个字段），依旧可以反序列化，但是如果数据类型变了，就不可以了，所以我们最好手动指定

## Parcelable
- 是android中独有的序列化方法，更适合android，效率高，但是使用麻烦，主要用在内存序列化上，如果是存储到设备上或者通过网络传输，推荐使用serializable
- parcel可以在binder中自由传输

## Binder

# 3. android进程间通讯的方式
## 使用Bundle 传输数据
- 四大组件中除了内容提供者，其他都支持通过Intent，携带Bundle传输数据，他也是实现了Parcelable接口

## 文件共享

```
序列化到本地

FileOutputStream fileOutputStream = openFileOutput("xuliehua.txt", MODE_PRIVATE);
Book book = new Book(10,100,"开发艺术");
ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
objectOutputStream.writeObject(book);
```
```
从本地反序列化

FileInputStream fileOutputStream = openFileInput("xuliehua.txt");
ObjectInputStream objectInputStream = new ObjectInputStream(fileOutputStream);
Book o = (Book) objectInputStream.readObject();
Log.e("MainActivity","send"+o.toString());
```

> 局限性：即并发读/ 写的问题。文件共享适合在对数据同步要求不高的进程之间进行通信。并且要妥善处理并发读写的问题。Sharedperfrences底层也是文件读写。

## Messenger 

>Messenger 是信使，通过它可以使Message在进程间传递

**服务端**

- 创建Service
- 构造Handler对象，实现handlerMessage方法。
- 通过Handler对象构造Messenger信使对象。  *new Messenger(handler);*
- 通过Service的onBind()返回信使中的Binder对象。*messenger.getBinder();*

**客户端**

- 创建Actvity
- 绑定服务
- 创建ServiceConnection,监听绑定服务的回调。
- 通过onServiceConnected()方法的参数，构造客户端Messenger对象。 *new Messenger(iBinder);*
- 通过Messenger向服务端发送消息。*messenger.send(message);*   客户端还可以构建一个handler，然后给Message设置replyTo，服务端就可以拿到客户端的Messenger了


## AIDL
AIDL是一种接口定义语言，用于约束两个进程间的通讯规则，供编译器生成代码，实现*Android设备*上的两个进程间通信(IPC)。

> 因为需要服务端和客户端共用aidl文件，所以最好单独建一个包，适合拷贝到客户端。

使用步骤
- 创建实体类（Book），实现Parcelable接口
- 创建AIDL包（as会自动创建：右键new->aidl即可）
- 创建实体类对应的aidl

```
//Book.aidl
package com.xk.niodemo;

parcelable BookAIDL;
```

- 创建interface（客户端将要调用的接口），注意，每个参数前面需要跟in、out、inout
  in：参数由客户端设置，或者理解成客户端传入参数值。
  out：参数由服务端设置，或者理解成由服务端返回值。
  inout：客户端输入端都可以设置，或者理解成可以双向通信。

```
// IBookManager.aidl
package com.xk.niodemo;

import com.xk.niodemo.BookAIDL;
interface IBookManager {
    List<BookAIDL> getBookList();
    void addBook(in BookAIDL book);
}

```
- 准备就绪，make project
- 然后会在\app\build\generated\source\aidl\debug\包名下生成文件
- 开始创建服务端
- 创建一个service
- 实现IBookManager.Stub这个类中的方法（也就是接口中的方法）
- 在onbinder中return 上一步实现的类
- 服务端在清单文件中注册，如果要被别的应用绑定，需要声明action
- 开始创建客户端
- 如果客户端和服务端在一个项目中，就不需要了，否则，将服务端的aidl目录拷贝在客户端
- bindService，在onServiceConnected中通过IBookManager.Stub.asInterface(service);可以获得IBookManager接口，然后就可以在客户端调用服务端的方法了


## ContentProvider
和Messenger一样，底层都是用的Binder


## Socket
java知识，不在赘述



## Android 进程间通信不同方式的比较

- Bundle:四大组件间的进程间通信方式，简单易用，但传输的数据类型受限。
- 文件共享：不适合高并发场景，并且无法做到进程间的及时通信。
- Messenger: 数据通过Message传输，只能传输Bundle支持的类型
- ContentProvider：android 系统提供的。简单易用。但使用受限，只能根据特定规则访问数据。
- AIDL:功能强大，支持实时通信，但使用稍微复杂。
- Socket：网络数据交换的常用方式。不推荐使用。