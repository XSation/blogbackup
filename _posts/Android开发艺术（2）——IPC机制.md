---
title: Android开发艺术（2）——IPC机制
date: 2017-09-10 19:14:22
tags:
categories: 阅读笔记
---

> 代码地址:https://github.com/kaikaixue/DevelopArtistical

# IPC简介

IPC就是进程间通讯的简称

线程是一种有限的资源

Windows：剪切板、管道、油槽等进行IPC

Linux：命名管道、共享内容、信号量等

Android：继承自Linux，但是通讯方式不完全继承Linux，最有特色的是Binder

> android一个应用可以开启多进程模式，为什么要开启？
>
> 1.android对每个应用的内存使用做了限制，比如16GB（当然现在版本不是了）用多进程可以扩大这个内存
>
> 2.可能某些模块需要运行在单独的进程等



# Android中的多进程

- 如果两个应用跑在同一个进程中，那么可以共享data目录、组件信息、内存等，或者说他们就像是同一个应用的两部分
- 清单文件中process可以指定全名也可以指定“：名称”，前者是全局进程，其他应用可以通过shareUID和他跑在同一个进程，后者是应用下的私有进程其他应用的组件不可以和他跑在同一进程
- P39这块理解有太多问题，有时间回来再看看，看是否可以理解



android会为每个应用程序，或者说每个进程分配一个独立的虚拟机，每个之间互不干扰，比如现在有com.xk.demo和com.xk.demo:remote，两个进程，他们同时访问Demo.class这个类，其实访问的是两个不同的对象，互不干扰的。

不同进程中的组件，通过内存共享数据就会失败

通常会有以下几种问题：

- 静态成员、单例模式失效
- 线程同步机制失效
- SharedPreferences可靠性下降
- Application多次创建

同一应用间的多进程其实就是相当于两个不同的应用使用shareUID的模式（毕竟Application创建了两次，所以说是两个应用也不为过）

共享内存不可行了，那么就要有一些别的方法来实现进程间通讯

# IPC基础概念

## Serializable

- 静态成员不属于类成员，不参与序列化
- transient修饰的成员变量不参与序列化
- 序列化后，会在文件中存储serializableid，反序列化的时候会和他进行比对，如果不一致，就crash，这就是为什么要写serializable的原因，防止类的结构改变后反序列化失败，当然修改了类名，字段类型等之后，无法反序列化了

## Parcelable

User实现了parcelable，User中有Book也实现了parcelable，那么在序列化User中的Book的时候需要传入当前线程上下文的classloader，否则会报找不到类的错误



二者的区别：Parcelable是android提供的，效率高，更适合android平台，但是使用起来麻烦，主要用于内存序列化中，在网络上传输之类的（进程间利用binder通信貌似需要的是Parcelable）。Serializable是java提供的，序列化的时候要做大量的io操作，会产生大量的临时变量，引起频繁的GC。在存储到本地的情况下，最好用serializable。

## Binder

简单的实现了一个aidl代码（[查看此处](https://github.com/kaikaixue/DevelopArtistical/blob/master/chapter2/src/main/java/com/xk/chapter2/server/IBookManagerStub.java)）

> 5.0之后需要显示调用service，不可以使用action、filiter的形式隐式启动

最恶心的binder来了。。。

```java
 /**
  * ICalculate是用来做计算功能的，那么他是在服务端的，他的实现（继承自Stub）也是在服务端的，而客户端需要调用它里面的方法，需要的就是这个接口对象，该方法的作用就是把Stub转换成可以在binder中传播的接口
  * 如果客户端和服务的在同一进程中，那么可以共享数据，所以直接返回这个Stub就好了，如果不在同一个进程中，就创建一个代理对象Stub.Proxy，并返回
  * Cast an IBinder object into an com.xk.chapter2.ICalculate interface,
  * generating a proxy if needed.
  */
public static com.xk.chapter2.ICalculate asInterface(android.os.IBinder obj) {
    //...Stub里面的asInterface方法
}
```

```java
/**
 * 该方法是服务端的方法（书上的话来说就是运行在Binder线程池，反正跟客户端不在一个进程中），只有在客户端、服务端不在一个进程的情况下才会走这里（如果在同一进程，asInterface会直接返回Stub对象，直接调用Stub里面的实现）
 * 如果不在同一进程，asInterface会返回代理对象（Stub.Proxy），当客户端调用方法的时候，内部会像下面一样，把客户端的参数写入到_data,它是一个parcel（parcelable可知，这个东西可以binder中传输），然后还有一个code（code标记了调用的是接口中的哪个方法）
 * 同时还有一个_reply,用来承载返回结果。然后_data、code、_reply等，这几个参数调用一个方法mRemote.transact(),很明显，这个是个远程方法，通过一系列底层实现，最后就会回调到服务端的这个onTransact方法（前面是在客户端，现在到了服务端了），然后服务端调用真实的方法实现，得到结果，写入
 * 到_reply中，这里有个返回值，返回false，那么客户端的请求就会失败，可以用此来对方法权限做限制，毕竟不是所有的客户端都可以调用服务端的
 * @param code
 * @param data
 * @param reply
 * @param flags
 * @return
 * @throws android.os.RemoteException
 */
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    //...Stub.onTransact()
}

```

说明一下：客户端线程中发起一个服务端请求，客户端线程就会被挂起，直到服务端返回数据，客户端才会继续执行，所以如果服务端的操作比较耗时，客户端应该放在子线程中去做。对于服务端，无论是否耗时，都不需要再开线程，而是要写成同步的，因为它本来就是运行在binder线程池中的。

![](/pic/androidkaifayishu2_1.png)

疑问：Proxy是如何实现两个进程通讯的 没看出来

在bindService的地方会创建一个ServiceConnection

```
ServiceConnection conn = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
        //这里的service，如果是在同一进程中，为Service的onBind中返回的那个类型，如果不是同一进程，这里返回的就是BindProxy，然后调用asInterface，拿到的就是对应的IBookManager了(Service的onBind返回的对象或者根据这个BindProxy对象创建的Proxy)
            iBookManager = IBookManagerStub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
```

### binder死亡代理

```
ServiceConnection conn = new ServiceConnection() {

    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        iBookManager = IBookManagerStub.asInterface(service);
        service.linkToDeath(recipient,0);
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
};
```

如上，在binder连接成功后，为它设置死亡代理，第一个参数是一个接口，当binder断开后，会回调这个接口，可以在接口里面做解除绑定的操作，然后继续重新bindService即可

# Android中的IPC方式

## Bundle

这个很简单，intent中可以传递它，它可以承载基本数据类型、Serializable、Parcellabled等

## 文件共享

Windows中读写文件会给文件加上排斥锁，使得其他线程无法访问。Linux不会，android基于Linux所以也一样，所以可以同时读写一个文件，但是很明显，这样会有问题发生。

记得大学时做一个App的数据缓存就用到了这个，一个进程中把一个对象序列化到本地，另一个进程中直接反序列化成对象即可。需要注意的是，这两个对象只是内容一样，本质上是两个不同的对象。

对于同步性要求不太高的情况，我们是可以通过共享文件来实现多进程通讯。SharedPreferences也是文件，但是他比较特殊，他是存储在包名目录下的，而且他会在内存中有缓存策略，所以在多进程并发操作的时候就不靠谱了，很大几率会丢失数据（多进程内存是不可以共享的），所以虽然他也是文件，但是极不推荐使用。

## Messenger

其实就是封装的AIDL,不过使用起来比较方便，传递的是Message对象，可以承载what、arg1、arg2、Bundle、replyTo等，他的obj属性在android2.2之前不支持跨进程传输，2.2之后也仅支持系统提供的Parcelable对象。Bundle可以支持很多数据类型，不过测试发现自定义的Parcelable好像也不行。具体看代码。

## AIDL

Messenger就是封装的AIDL，他只适合简单的消息传递，如果有大量的并发请求，或者调用函数，就不合适了。

AIDL支持：

- 基本数据类型
- String和CharSequence
- List：支持吃ArrayList，并且每个元素需要被AIDL支持（注意，没了良好的控制并发读写，在服务端可以用CopyOnWriteArrayList代替ArrayList，他不是继承自ArrayList，但是不会报错，因为AIDL接受的是List接口，在传到客户端的时候，CopyOnWriteArrayList会被以ArrayList的形式转换出去）
- Map：支持HashMap，key、value都需要被AIDL支持
- Parcelable：实现了Parcelable的对象（使用的时候不论是否在同一个包内，都需要显示的import,而且还要新建一个同样名字的aidl文件）
- AIDL自身的接口（这个在demo中用了，可以看代码）

除了基本数据类型，都要加上in、out、inout

aidl接口只支持方法，不支持生命就静态变量

客户端调用服务端方法后，会等待服务端方法执行完，服务端方法在Binder线程池中，所以如果服务端方法耗时的话，客户端就会被挂起，所以客户端要注意时间太长要开线程

#### 关于断开重连：

- 设置死亡代理，前面讲过（UI线程中回调）
- 在ServiceConnection的回调中设置（客户端的binder线程池中回调）

#### 关于监听器：

对象在多进程中是无法传输的（内存不共享），对象的传输本质是序列化和反序列化（实现parcelable的原因），所以当客户端设置一个监听器对象给服务端的时候，服务端解除这个监听时，对象不是一个，所以可以用系统提供的类RemoteCallbackList代替普通的List去维护监听器的集合

#### 关于权限控制

- service的onbind中校验权限，然后返回binder对象或者null（onbind运行在服务端的UI线程中，不是一个binder调用，只能验证服务端的权限，所以没意义，所以不用了）
- Stub.onTransact中返回false表示拒绝客户端请求，具体看demo



**以上任何疑问直接看demo**



## ContentProvider

### 关于清单文件中的注册

```
<!--内容提供者必须要注册，属于四大组件-->
<!--authorities 匹配这个提供者的uri算是前缀吧-->
<!--可以指定权限，也可以指定读权限、写权限（增删改查 哪个读 哪个写 很清楚）测试发现同一个app中两个进程权限控制没用?或许放在两个app就好了，懒得测试了-->
<!--一般内容提供者用在多进程通讯中，所以我这里为provider指定了一个单独的进程，provider也可以放在其他app中，效果是一样的-->
<provider android:name=".provider.BookProvider"
			android:authorities="com.xk.chapter2.provider.BookProvider"
            android:process=".bookprovider"
            android:readPermission="read"
            android:writePermission="write" />
```

###  关于Provider的定义
注意，内容观察者底层也是通过binder实现的，但是比aidl用起来简单，他仅仅是帮助我们实现了进程间通讯，具体的数据存储、查询之类的 还是需要我们自己完成的，可以使用sqlite、sp、甚至文件 各种都无所谓，但是可能被多个进程、多个线程同时调用（想想通讯录），所以对于线程同步需要考虑，如果用的是一个sqlitehelper的话，系统以及实现了，总之视情况而定吧

oncreate是被系统调用的，运行在主线程。其他几个方法是运行在binder线程池中，打印日志即可知道，代码中已经写好，详情见代码	

在静态代码块中指定一些匹配规则，如下代码，然后在增删改查方法中就可以匹配了

```
static UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

static {
    //在匹配规则中添加两条规则：uri后面的path是book的时候，可以匹配为0，user时可以匹配为1,然后在增删改查中把uri转换成对应的0、1
    uriMatcher.addURI("com.xk.chapter2.provider.BookProvider", "book", 0);
    uriMatcher.addURI("com.xk.chapter2.provider.BookProvider", "user", 1);
}

@Nullable
@Override
public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
    int uriCode = uriMatcher.match(uri);
    String tableName = getTableName(uriCode);//根据匹配的code，返回对应的文件名、数据库名等
    // TODO: by xk 2017/9/17 15:37 这里操作数据库、文件等查询操作
    Log.d("BookProvider", "query-->" + Thread.currentThread());
    return null;
}
```

#### 观察者模式

数据内容改变之后，可以利用notifyChange通知观察者，这个想想手机通讯录即可

```
@Nullable
@Override
public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
    Log.d("BookProvider", "insert-->" + Thread.currentThread());
    // TODO: by xk 2017/9/17 15:37 参照query，找到操作的表，然后存数数据
    getContext().getContentResolver().notifyChange(uri, null);//通知观察者，这个uri下的数据被改变了
    return null;
}
```

#### 自定义调用方法

使用call可以实现自定义方法调用，具体的使用方式是很灵活的，我随便写个demo，不一定准确，就是一种思路，不过一般也不这么写，随意发挥即可

```
//除了增删改查，还可以通过call方法实现远程进程中方法的调用，这么一来不就是跟aidl实现的效果一样了吗，还很简单，来试一试（这里只是我瞎猜的写的一种使用办法，随机应变喽）(注意该方法是在binder线程池中，阻塞没关系，但是他的调用者——客户端就需要注意了)
@Nullable
@Override
public Bundle call(@NonNull String method, @Nullable String arg, @Nullable Bundle extras) {
    Bundle bundle = new Bundle();

    String[] as = arg.split("a");
    if (method.equals("add")) {
        int add = add(Integer.parseInt(as[0]), Integer.parseInt(as[1]));
        bundle.putInt("result", add);

    } else if (method.equals("sub")) {
        int add = sub(Integer.parseInt(as[0]), Integer.parseInt(as[1]));
        bundle.putInt("result", add);
    }
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return bundle;
}


private int add(int a, int b) {
    return a + b;
}

private int sub(int a, int b) {
    return a - b;
}
```

这是provider中的代码 ，在客户端直接像调用增删改查一样，调用call即可，三个参数的含义直接看上面这个方法就行

### 关于Provider的使用

以下是在客户端的代码，如果provider中声明了权限，记得在清单文件中声明，测试发现，如果客户端、服务端是同一个app的不同进程，不需要声明权限（猜测：毕竟是同一个app，权限展示也是以app为基准的吧）

- 查询

  ```
  Uri uri = Uri.parse("content://com.xk.chapter2.provider.BookProvider");
  Log.d("ProviderActivity", "onClick-->" + Thread.currentThread());
  getContentResolver().query(uri, null, null, null, null);
  ```

- call

  ```
  Uri uri3 = Uri.parse("根据需求写");
  Log.d("ProviderActivity", "onClick-->" + Thread.currentThread());
  Bundle add = getContentResolver().call(uri3, "sub", "1a2", new Bundle());
  Toast.makeText(this, add.getInt("result")+"", Toast.LENGTH_SHORT).show();
  ```

uri的话，后面加path，然后在provider中解析后可知要访问的数据（是book还是user），path后面还可以加别的参数，其实这只是一个约定的规范而已，随便传，然后字符串解析应该也没问题吧，不过估计没人那么干，会被同事骂的。

## Socket

> 服务的readLine，那么客户端需要writeLine，或者末尾加\n，这个折腾了好久，之前一直发不过去数据

# Binder连接池

一个aidl对应一个service，很明显是不好的，毕竟service是重量级的，在设置的app详情中也是可以看到的，所以如果写一个Binder连接池，通过code直接返回对应的IBinder，最好了，详情见代码

# 选择合适的IPC方式

![](/pic/ipc_ways.png)