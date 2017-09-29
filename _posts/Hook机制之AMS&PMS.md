---
title: Hook机制之AMS&PMS
categories: DroidPlugin
date: 2017-04-16 

---
> 参考资料 :[Android 插件化原理解析——Hook机制之AMS&PMS](http://weishu.me/2016/03/07/understand-plugin-framework-ams-pms-hook/) 



## AMS&PMS的重要作用
- AMS（ActivityManagerService），四大组件无一不与他打交道
- `startActivity`最终调用的是AMS的startActirivy
- `startService`、`bindService`都是调用的AMS的startService、bindService
- 动态广播的注册和接受都是在`AMS`中完成(静态广播在`PMS`中完成)
- `getContentResolver`最终从AMS中的`getContentProvider`中获取`ContentProvider`
- PMS(PackageManagerService)
- 权限校验(`checkPermission,checkUidPermission`)
- apk的mate信息获取(`getApplicationInfo`)
- 四大组件的信息获取(`query`系列方法)等重要功能

## AMS获取过程

前面说了`startActivity`最终调用的是AMS的startActirivy,那么先来说说startActivity

#### startActivity的两种方式:
1. **context.startActivity()** 
    这样启动的activity没有任务栈,需要设置`FLAG_ACTIVITY_NEW_TASK`这个flag
2. **activity.startActivity()**
    通常在activity中启动就是这种模式


### Context.startActivity()
context的实现类是contextImpl,那就看它
```
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();
        if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
```
我们知道了两点
1. 为什么要设置flag
2. 最终执行的是`Instrumentation`中的`execStartActivity`方法
   跟踪进去
```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    //...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        //i am here ^_^
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```
最后执行的是`ActivityManagerNative.getDefault().startActivity();`,所以启动Activity是在这里干的

`ActivityManagerNative`就是本地Binder,相当于Stub,不过他是一个抽象类,具体的实现在`ActivityManagerService`,`ActivityManagerNative`中还有一个代理.而`IActivityManager`就是IInterface. 而`ActivityManager`只是一个管理类而已.
### Activity.startActivity
翻翻源码可以发现,他最后也是到了ActivityManagerNative,所以跟Context启动的并无本质区别

所以启动Activity最后都是通过Instrumentation,从而调用了ActivityManagerNative的方法,最终通过ActivityManagerNative调用了AMS(ActivityManagerService)的方法. `ActivityManagerNative.getDefault()`就是代理,通过IPC和ActivityManagerService通讯.


具体看看ActivityManagerNative.getDefault()
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
其实就是拿到一个IInterface,就可以干一些事了,而且是被保存到单例模式中(由于需要与AMS进行频繁的通讯,所以把它保存成单例,这也就是我们不用Binder Hook去hook AMS,因为我们只需要替换这个单例即可)

```
private void hookAMS() {
        try {
            Class<?> activityManagerNative = Class.forName("android.app.ActivityManagerNative");
            Field gDefault = activityManagerNative.getDeclaredField("gDefault");
            gDefault.setAccessible(true);
            //获取默认的Singleton<IActivityManager>
            Object original = gDefault.get(null);

            Class<?> singletonClass = Class.forName("android.util.Singleton");
            Field mInstance = singletonClass.getDeclaredField("mInstance");
            mInstance.setAccessible(true);

            //iActivityManager
            Object  iActivityManager = mInstance.get(original);

            //hook iActivityManager
            iActivityManager=hookIActivityManager(iActivityManager);



            mInstance.set(original,iActivityManager);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }


    }

    private Object hookIActivityManager(Object iActivityManager) {
        IActivityManagerHandler invocationHandler=new IActivityManagerHandler(iActivityManager);
        return Proxy.newProxyInstance(iActivityManager.getClass().getClassLoader(),iActivityManager.getClass().getInterfaces(), invocationHandler);
    }



}
class IActivityManagerHandler implements InvocationHandler {
    private static final String TAG = "MainActivity";
    private Object base;

    public IActivityManagerHandler(Object base) {
        this.base = base;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        Log.e(TAG,"invoke==============>"+method.getName());
        //可以在这里针对不同的方法进行AOP(执行前/执行后加代码,或者直接干脆hook掉base)
        return  method.invoke(base,objects);
    }
```


对于startService,bindService等等方法,跟踪源码发现,他们最总都是到了`ActivityManagerNative.getDefault()`,所以hook了这个`gDefault`,所有和AMS(上述代码不就是替换了`IActivityManager`吗)交互的入口就都被hook了



## PMS获取过程

有了前面的基础,PMS就很简单了,直接查看ContextImpl的getPackageManager,不过他用ApplicationPackageManager包装了一次.

我们只需要hook如下两处即可
1. ActivityThread的静态字段sPackageManager
2. 通过Context类的getPackageManager方法获取到的ApplicationPackageManager对象里面的mPM字段。

```
// 获取全局的ActivityThread对象
Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
Object currentActivityThread = currentActivityThreadMethod.invoke(null);

// 获取ActivityThread里面原始的 sPackageManager
Field sPackageManagerField = activityThreadClass.getDeclaredField("sPackageManager");
sPackageManagerField.setAccessible(true);
Object sPackageManager = sPackageManagerField.get(currentActivityThread);

// 准备好代理对象, 用来替换原始的对象
Class<?> iPackageManagerInterface = Class.forName("android.content.pm.IPackageManager");
Object proxy = Proxy.newProxyInstance(iPackageManagerInterface.getClassLoader(),
        new Class<?>[] { iPackageManagerInterface },
        new HookHandler(sPackageManager));

// 1. 替换掉ActivityThread里面的 sPackageManager 字段
sPackageManagerField.set(currentActivityThread, proxy);

// 2. 替换 ApplicationPackageManager里面的 mPM对象
PackageManager pm = context.getPackageManager();
Field mPmField = pm.getClass().getDeclaredField("mPM");
mPmField.setAccessible(true);
mPmField.set(pm, proxy);
```


> 通过之前的练习,算是真的理解了:单例或者静态变量是最容易hook的,相反,普通成员变量就比较麻烦了,因为我们需保证每一个对象中的变量都被hook掉


