---
title: Hook机制之Binder Hook
categories: DroidPlugin
date: 2017-04-15 

---
> 参考资料 :[Android插件化原理解析——Hook机制之Binder Hook](http://weishu.me/2016/02/16/understand-plugin-framework-binder-hook/) 
> 阅读前先掌握[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

## 什么是Binder Hook

hook系统服务就叫`Binder Hook`,因为Linux的特性决定了，各个进程间不能直接通讯，系统服务提供了进程间通讯的可能，那么系统服务一定是存在于各个进程（用户）中的Binder，所以hook系统服务就叫`Binder Hook`

## 如何获取一个系统服务

想要hook系统服务，肯定要知道他是怎么来的，那么就开始看源码

我们通常获取服务是这样干的`context.getSystemService("SERVICE_NAME")`

Context的默认实现是ContextImpl，`getSystemService`在`ContextImpl`中是这样的
```java
@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```
继续往下看`SystemServiceRegistry`
```java
/**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```

在`SystemServiceRegistry`中，从一个集合`SYSTEM_SERVICE_FETCHERS`中获取服务，这个集合是在`SystemServiceRegistry`的静态代码块中初始化的，只看两个比较典型的`ACTIVITY_SERVICE`和 `ACCOUNT_SERVICE`

```
 registerService(Context.ACCOUNT_SERVICE, AccountManager.class,
                new CachedServiceFetcher<AccountManager>() {
            @Override
            public AccountManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ACCOUNT_SERVICE);
                IAccountManager service = IAccountManager.Stub.asInterface(b);
                return new AccountManager(ctx, service);
            }});

registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});

```
- 先分析`AccountManager`,它通过`ServiceManager`拿到了一个IBinder,然后将其转换成了`IInterface`(看不懂先看Weishu大神的[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)),IInterface决定了它可以做什么,于是就创建了`AccountManager`
- 再分析ActivityManager,可以看到,他并没有调用`ServiceManager`,其实不然,前面的`AccountManager`直接就可以看到,它是从`ServiceManager`中获取的,那么为什么`AccountManager`不直接从`ServiceManager`获取呢,是因为`ActivityManager`的实现其实都是由`ActivityManagerNative.getDefault()`去完成的,通过此方法获得一个`IInterface`,那么就看这个`ActivityManagerNative.getDefault()`是怎么做的
```
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```
所以就证实了各种系统服务都是通过`ServiceManager.getService("SERVICE_NAME")`获取的

综上所述,系统服务的获取就是两步:
```
IBinder iBinder = ServiceManager.getService("SERVICE_NAME");//获取IBinder
IXXXManager iManager=IXXXManager.Stub.asInterface(iBinder);//转换为Service,这个service也就是IInterface
```
## 寻找hook点

首先想一下我们是要干什么?我们是想hook掉系统服务,也就是替换掉前文中的各种`IXXManager`,改变它的默认实现,比如hook掉`ClipboardManager`(它就是`IXXManager`),让我们在粘贴一个内容的时候,改变粘贴的内容.也就是让上面的`IXXXManager iManager=IXXXManager.Stub.asInterface(iBinder);`中的返回值改变,为了简单起见,写一个简单的aidl代码,然后分析一下它的这个方法.
```
        /**
         * Cast an IBinder object into an com.xk.aidl.IComputer interface,
         * generating a proxy if needed.
         */
        public static com.xk.aidl.IComputer asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.xk.aidl.IComputer))) {
                return ((com.xk.aidl.IComputer) iin);
            }
            return new com.xk.aidl.IComputer.Stub.Proxy(obj);
        }
```
首先从本地查找,如果有就直接return,否则创建一个`Proxy`(这个`Proxy`是本地service)

那么我们就让这个` android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);`作为hook点,只要这我们返回一个`IInterface`,并且让他满足下面的判断,就达到我们的目的了
```
if (((iin != null) && (iin instanceof com.xk.aidl.IComputer))) {
                return ((com.xk.aidl.IComputer) iin);
}
```
所以此时,我们需要拿到上面的这个`obj`,并且hook它,改造它,`obj`就是`IBinder`,可以通过`IBinder iBinder = ServiceManager.getService("SERVICE_NAME");`拿到,所以这里只需要修改`getService`的返回值即可.(注意,这里的IBinder也仅仅是存在于用户进程中的代理`BinderProxy`)

`getService("SERVICE_NAME")`是一个静态方法,如果它仅仅是获取service,那我们是没法搞的,因为我们没法拦截一个静态方法([Android插件化原理解析——Hook机制之动态代理](http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)中提到,hook最好选择一个静态变量或者单例),那么我们看看具体实现(`ServiceManager`是一个隐藏类,所以从[这里](http://www.grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/os/ServiceManager.java#ServiceManager)去看)



```
public static IBinder getService(String name) {
         try {
             IBinder service = sCache.get(name);
             if (service != null) {
                 return service;
             } else {
                 return getIServiceManager().getService(name);
             }
         } catch (RemoteException e) {
             Log.e(TAG, "error in getService", e);
         }
         return null;
     }
```
它会先从缓存中拿,所以我们只需要修改缓存中的数据就好了(由于取`IBinder`对象,准确的说是`BinderProxy`,需要跨进程访问,为了减少跨进程次数,使用了缓存机制)

----------


 **总结一下:我们的目的是hook系统服务,所以我们先写一个伪造的系统服务,比如`ClipboardManager`,使得我们在调用`context.getSystemService("SERVICE_NAME")`时返回的是这个伪造的服务.而这个方法通过上面分析,最后走到了**
```
IBinder iBinder = ServiceManager.getService("SERVICE_NAME");
IClipboardManager iManager=IClipboardManager.Stub.asInterface(iBinder);
```
**看第二行代码的内部实现,我们只需要hook掉iBinder的`queryLocalInterface`方法即可,最后的问题就是修改第一行中getService的返回值,再进去看,其实就是修改`ServiceManager`中的缓存`sCache`中的值**

## hook系统能够剪切板服务

```
            //通过反射，拿到ServiceManager
            Class<?> serviceManagerClass = Class.forName("android.os.ServiceManager");
            Method getService = serviceManagerClass.getDeclaredMethod("getService", String.class);
            getService.setAccessible(true);
            //这个iBinder是hook之前的binder对象
            IBinder iBinder = (IBinder) getService.invoke(null, Context.CLIPBOARD_SERVICE);
            //hook IBinder，使他的queryLocalInterface的返回值始终存在，并且在asInterface中直接返回，而不是让系统去new一个代理，这样，返回的内容就可以由我们决定了
            IBinder hookIBinder = hookIBinder(iBinder);

            //把hookIBinder存入到sCache中，这样serviceManager可以直接取到这个修改过IBinder，而不是从远程获取
            Field sCache = serviceManagerClass.getDeclaredField("sCache");
            sCache.setAccessible(true);
            HashMap sCacheSource = (HashMap) sCache.get(null);
            sCacheSource.put(Context.CLIPBOARD_SERVICE, hookIBinder);
            sCache.set(null, sCacheSource);

```

hook IBinder的handler
```
//修改queryLocalInterface的返回值
class IBinderInvocationHandler implements InvocationHandler {
    private static final String TAG = "IBinderInvocationHandle";
    IBinder base;
    private  Class<?> stubClass=null;
    private  Class<?> iClipboardClass=null;
    private  Object iClipboard;

    public IBinderInvocationHandler(IBinder iBinder) {
        this.base = iBinder;
        try {

            stubClass = Class.forName("android.content.IClipboard$Stub");

            Method asInterface = stubClass.getDeclaredMethod("asInterface", IBinder.class);

            iClipboard = asInterface.invoke(null, iBinder);
            iClipboardClass = Class.forName("android.content.IClipboard");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

    }
    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        if (method.getName().equals("queryLocalInterface")) {
            //这里应该返回一个android.os.IInterface,并且满足 if (((iin != null) && (iin instanceof android.content.IClipboard)))
//
            //这里要创建一个自己的IClipboard
            IClipboardInvocationHandler clipoardHandler=new IClipboardInvocationHandler(base,iClipboard);
            return Proxy.newProxyInstance(iClipboardClass.getClassLoader(),iClipboard.getClass().getInterfaces(),clipoardHandler);
//            return proxy.queryLocalInterface((String) objects[0]);
        } else {
            Log.e(TAG, "没进来======================> " + method.getName());
            return method.invoke(base, objects);
        }
    }

```

hook IClipboard的Handler
```
/**
 * 修改Clipboard的getPrimaryClip和hasPrimaryClip方法
 */
class IClipboardInvocationHandler implements InvocationHandler {
    private IBinder iBinder;
    private Object iClipboard;
    public IClipboardInvocationHandler(IBinder iBinder,Object iClipboard) {
        this.iBinder = iBinder;
        this.iClipboard=iClipboard;
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        if (method.getName().equals("getPrimaryClip")) {
            Method getPrimaryClip = iClipboard.getClass().getDeclaredMethod("getPrimaryClip", String.class);
            ClipData clipData = (ClipData) getPrimaryClip.invoke(iClipboard, objects);
           return  ClipData.newPlainText(null,clipData.getItemAt(0).getText()
                   +" success!!!");
        }

//        if (method.getName().equals("hasPrimaryClip")) {
//            return false;
//        }
        return method.invoke(iClipboard,objects);
    }
}
```


