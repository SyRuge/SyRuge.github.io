## Activity启动流程(MainActivity->SecondActivity)

在描述之前，需要了解相关概念。

### ApplicationThread

在ActivityThread内部，继承与ApplicationThreadNative，实际上是一个Binder对象。AMS最终会拿到这个实例，通过它就可以跟Activity进行交互。

### ActivityThread

Activity内部有一个成员变量mMainThread，其类型为ActivityThread，用来描述一个应用程序**进程**。系统每启动一个应用程序进程时，就会在它里面创建一个**ActivityThread**实例，并且会将这个实例保存在该进程每一个被启动Activity的启动Activity的**mMainThread**中(A启动B，保存在A中)。

### ActivityRecord

每一个启动的Activity在AMS中都有一个对应的**ActivityRecord**，用来维护对应的Activity的运行状态以及信息。

#### Activity#startActivityForResult

startActivity最终会调用Activity的startActivityForResult方法：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        // Instrumentation是用来监控程序与系统之间的交互操作的，
        // mMainThread中。ActivityThread的成员函数getApplicationThread用来获取它内部一个类型为
        // ApplicationThread对象作为参数传递给变量mInstrumentation的成员函数execStartActivity，
        // 以便将它传递给AMS，这样就可以通过ApplicationThread与Activity交流了。
        // mToken的类型为IBinder，它是Binder代理对象，指向AMS中一个类型为ActivityRecord对象。
        // 此处传入也是为了传递给AMS，这样AMS就可以得到Activity的详细信息了。
        // startActivity启动的requestCode为-1
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
    }
}
```

#### Instrumentation#execStartActivity

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    try {
        // 获取AMS的代理对象AMP（ActivityManagerProxy），然后调用起startActivity方法，通过该方法
        // 通知AMS启动Activity
        // whoThread为ApplicationThread
        // requestCode=-1 option=null
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        // 检查启动Activity的结果
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
// ActivityManagerNative
static public IActivityManager getDefault() {
        return gDefault.get();
}
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            // 通过ServiceManager获取一个名字为activity的服务代理对象，即获取一个引用了AMS的代理对象。
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            // 调用asInterface方法将这个代理对象封装成一个类型为AMP的代理对象，最后将其保存在静态成员
            // 变量gDefault中，所以可以通过get方法直接获取
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
  };
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in =
        (IActivityManager) obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    return new ActivityManagerProxy(obj);
}
```

ActivityManagerNative.getDefault()实际上是获取了一个AMS的代理对象ActivityManagerProxy。

#### ActivityManagerNative.ActivityManagerProxy#startActivity

```java
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
            case START_ACTIVITY_TRANSACTION: {
                IApplicationThread app = ApplicationThreadNative.asInterface(b);
                int result = startActivity(app, callingPackage, intent, resolvedType,
                        resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
                reply.writeNoException();
                reply.writeInt(result);
                return true;
            }
}
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
                         String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    // caller指启动的Activity的ApplicationThread(A启动B指A的ApplicationThread)
    // intent包含了被启动的Activity的相关信息
    // resultTo是AMS的一个ActivityRecord对象，保存了A的先关信息，注意他是一个Binder
    // 通过AMP类内部的一个IBinder代理对象向AMS发送一个类型为START_ACTIVITY_TRANSACTION的进程间通信请求
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}
```

写入相关数据，通过AMP内部的一个Binder代理对象mRemote发起一个START_ACTIVITY_TRANSACTION的进程间通信请求。终点关注三个参数：caller、intent和resultTo。

#### ActivityManagerService#startActivity

上述步骤完成后，进入ActivityManagerService执行。startActivity用来处理START_ACTIVITY_TRANSACTION进程间通信请求。

```java
public final int startActivity(IApplicationThread caller, String callingPackage,
                               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

```java
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                                     Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                     int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    // mActivityStarter是Activity启动等操作的管理者
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, null);
}
```

#### ActivityStarter#startActivityMayWait

```java
// 该函数的Wait表示对于outResult的处理上，我们启动Activity时inTask为空
final int startActivityMayWait(IApplicationThread caller, int callingUid, String callingPackage, Intent intent, String resolvedType, IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config, Bundle bOptions, boolean ignoreTargetSecurity, int userId, IActivityContainer iContainer, TaskRecord inTask) {
    // 被启动的Activity是第一次启动时，是没有设置标签的，所以下面不会走，如果被启动的Activity已经存在了，
    // 并且被设置了下面标签就会执行。PRIVATE_FLAG_CANT_SAVE_STATE是用于声明App是否享受系统提供的
    // Activity状态保存/恢复功能的。但是似乎没有App能成为heavy-weight process，因为PackageParser的
    // parseApplication方法并不会解析该标签。设置这个属性主要是为了软件的流程性
            if (aInfo != null &&
                    (aInfo.applicationInfo.privateFlags
                            & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            }
    final ActivityRecord[] outRecord = new ActivityRecord[1];
            int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                    aInfo, rInfo, voiceSession, voiceInteractor,
                    resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                    options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                    inTask);
}
// Locked 代表非线程安全的，提醒我们必须保证这些函数是线程安全的，（因为他们涉及不可重入资源的处理）
 final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent, String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo, IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid, String callingPackage, int realCallingPid, int realCallingUid, int startFlags, ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container, TaskRecord inTask) {
     
 }
```

