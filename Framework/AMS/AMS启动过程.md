# AMS启动过程

 分析Android系统服务ActivityManagerService，简称AMS

```
frameworks/base/core/java/android/app/
  - ActivityThread.java
  - LoadedApk.java
  - ContextImpl.java
frameworks/base/services/java/com/android/server/
  - SystemServer.java
frameworks/base/services/core/java/com/android/server/
  - SystemServiceManager.java
  - ServiceThread.java
  - pm/Installer.java
  - am/ActivityManagerService.java
```

## 一、概述

讲述system_server进程中AMS服务的启动过程，以[frameworks/base/services/java/com/android/server/SystemServer.java] main函数为起点， new SystemServer().run() 展开进行分析

依次执行startBootstrapServices()，startCoreServices()，startOtherServices() 3个方法

```java
private void run() {
    ...
    // Start services.
    try {
        t.traceBegin("StartServices");
        startBootstrapServices(t);
        startCoreServices(t);
        startOtherServices(t);
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        t.traceEnd(); // StartServices
    }
    ...
}
```

## 二. AMS启动

### 2.1 startBootstrapServices [-> SystemServer.java]

通过SystemServiceManager.startService() 启动 ActivityManagerService.Lifecycle

```java
private void startBootstrapServices() {
    ...
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    //设置AMS的系统服务管理器
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    //设置AMS的APP安装器
    mActivityManagerService.setInstaller(installer);
    //初始化AMS相关的PMS
    mActivityManagerService.initPowerManagement();
    //注册各种系统服务
    mActivityManagerService.setSystemProcess();
    ...
}
```

#### 2.1.1 AMS.Lifecycle [-> ActivityManagerService.java]

- 创建ActivityManagerService.Lifecycle对象。
- 调用Lifecycle.onStart()方法。

```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        //创建ActivityManagerService
        mService = new ActivityManagerService(context, sAtm);
    }

    @Override
    public void onStart() {
        mService.start(); 
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```

#### 2.1.2 AMS创建 [ActivityManagerService.java]

- 创建了3个线程，分别为”ActivityManager”，”android.ui”，”CpuTracker”。

- 创建广播队列（前台广播，后台广播，offload广播）

- 创建/system目录

- 创建ActivityTaskManger

```java
public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm)) {
    mContext = systemContext;
    mInjector = new Injector(systemContext);

    mFactoryTest = FactoryTest.getMode();//默认为FACTORY_TEST_OFF
    mSystemThread = ActivityThread.currentActivityThread();

    //创建名为"ActivityManager"的前台线程，并获取mHandler
    mHandlerThread = new ServiceThread(TAG, THREAD_PRIORITY_FOREGROUND, false);
    mHandlerThread.start();

    mHandler = new MainHandler(mHandlerThread.getLooper());
    //创建名为"android.ui"的线程
    mUiHandler =  mInjector.getUiHandler(this);
    //前台广播接收器，在运行超过10s将放弃执行
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", BROADCAST_FG_TIMEOUT, false);
    //后台广播接收器，在运行超过60s将放弃执行
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);
    mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;
    mBroadcastQueues[2] = mOffloadBroadcastQueue;
    //创建ActiveServices，其中非低内存手机mMaxStartingBackground为8
    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);

    //创建目录/data/system
    File systemDir = SystemServiceManager.ensureSystemDir();;

    //创建服务BatteryStatsService
    mBatteryStatsService = new BatteryStatsService(systemDir, mHandler);
    mBatteryStatsService.getActiveStatistics().readLocked();
    ...

    //创建进程统计服务，信息保存在目录/data/system/procstats，
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
    mAppOpsService = new AppOpsService(new File(systemDir, "appops.xml"), mHandler);
    mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"));

    // User 0是第一个，也是唯一的一个开机过程中运行的用户
    mStartedUsers.put(UserHandle.USER_OWNER, new UserState(UserHandle.OWNER, true));
    mUserLru.add(UserHandle.USER_OWNER);
    updateStartedUserArrayLocked();
    ...
    // 创建 ActivityTaskManger
    mActivityTaskManager = atm;
    mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());

    //创建名为"CpuTracker"的线程
    mAppProfiler = new AppProfiler(this, BackgroundThread.getHandler().getLooper(), null
        }
    };
    ...
}
```

#### 2.1.3 AMS.start[ActivityManagerService.java]

- 移除所有进程组

- 添加LocalService到集合LocalServices

```java
private void start() {
    Process.removeAllProcessGroups(); //移除所有的进程组

    mBatteryStatsService.publish(mContext); //启动电池统计服务
    mAppOpsService.publish(mContext);
    //创建LocalService，并添加到LocalServices
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    // 启动CpuTracker    
    mAppProfiler.onActivityManagerInternalAdded();
}
```

#### 2.1.4 AMS.setSystemProcess [-> startBootstrapServices]

该方法主要工作是注册各种服务

```java
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this));
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
        synchronized (this) {
            //创建ProcessRecord对象
          ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                        false,
                        0,
                        new HostingRecord("system"));
            app.persistent = true; //设置为persistent进程
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);//维护进程lru
            updateOomAdjLocked(); //更新adj
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException("", e);
    }
}
```

#### 2.1.5 AT.installSystemApplicationInfo [-> setSystemProcess -> ActivityThread.java]

该方法调用ContextImpl的installSystemApplicationInfo()方法，最终调用LoadedApk的installSystemApplicationInfo，加载名为“android”的package

```java
public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    synchronized (this) {
        getSystemContext().installSystemApplicationInfo(info, classLoader);
        mProfiler = new Profiler();    //创建用于性能统计的Profiler对象
    }
}
```

#### 2.1.6 installSystemApplicationInfo [-> LoadedApk.java]

```java
void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    assert info.packageName.equals("android");
    mApplicationInfo = info; //将包名为"android"的应用信息保存到mApplicationInfo
    mClassLoader = classLoader;
}
```

### 2.2 startOtherServices [-> SystemServer.java]

```java
private void startOtherServices() {
  ...
  //安装系统Provider
  mActivityManagerService.installSystemProviders();
  ...
  //After receiving this boot phase, services can obtain lock settings data.
  //phase 480 
  mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
  //After receiving this boot phase, services can safely call into core system services
  //such as the PowerManager or PackageManager.
  //phase 500
  mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);
  ...
  mActivityManagerService.systemReady(() -> {
    //After receiving this boot phase, services can broadcast Intents.
    //phase 550
    mSystemServiceManager.startBootPhase(SystemService.PHASE_ACTIVITY_MANAGER_READY);
    ...
    //After receiving this boot phase, services can start/bind to third party apps.
    //Apps will be able to make Binder calls into services at this point.
    //phase 600
    mSystemServiceManager.startBootPhase(SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
    ...
  }
}
```

#### 2.2.1 AMS.installSystemProviders[-> ContentProviderHelper.java]

```java
public final void installSystemProviders() {
    List<ProviderInfo> providers;
    synchronized (this) {
        ProcessRecord app = mProcessNames.get("system", Process.SYSTEM_UID);
        providers = generateApplicationProvidersLocked(app);
        if (providers != null) {
            for (int i=providers.size()-1; i>=0; i--) {
                ProviderInfo pi = (ProviderInfo)providers.get(i);
                //移除非系统的provider
                if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                    providers.remove(i);
                }
            }
        }
    }
    if (providers != null) {
        //安装所有的系统provider
        mSystemThread.installSystemProviders(providers);
    }
    // 创建核心Settings Observer，用于监控Settings的改变。
    mCoreSettingsObserver = new CoreSettingsObserver(this);
}
```

## 三. AMS的systemReady

### 3.1 startOtherServices [-> SystemServer.java]

```java
private void startOtherServices() {
  ...
  mActivityManagerService.systemReady(()-> {
      //phase550
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_ACTIVITY_MANAGER_READY);

      mActivityManagerService.startObservingNativeCrashes();
      //启动WebView
      WebViewFactory.prepareWebViewInSystemServer();
      //启动System UI
      startSystemUi(context);
      // 执行一系列服务的systemReady方法
      networkScoreF.systemReady();
      networkManagementF.systemReady();
      networkStatsF.systemReady();
      networkPolicyF.systemReady();
      connectivityF.systemReady();
      audioServiceF.systemReady();
      Watchdog.getInstance().start(); //Watchdog开始工作
      //phase600
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
      //执行一系列服务的systemRunning方法
      wallpaper.systemRunning();
      inputMethodManager.systemRunning(statusBarF);
      location.systemRunning();
      countryDetector.systemRunning();
      networkTimeUpdater.systemRunning();
      commonTimeMgmtService.systemRunning();
      textServiceManagerService.systemRunning();
      assetAtlasService.systemRunning();
      inputManager.systemRunning();
      telephonyRegistry.systemRunning();
      mediaRouter.systemRunning();
      mmsService.systemRunning();
  });
}
```

#### 3.1.1 startSystemUi

启动服务”com.android.systemui/.SystemUIService”

```java
static final void startSystemUi(Context context) {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName("com.android.systemui",
                "com.android.systemui.SystemUIService"));
    context.startServiceAsUser(intent, UserHandle.OWNER);
}
```

### 3.2 AMS.systemReady() [-> SystemServer.java]

#### 3.2.1 该阶段的主要功能：

- 杀掉procsToKill中的进程, 杀掉进程且不允许重启；
- 此时，系统和进程都处于ready状态；

```java
synchronized(this) {
    if (mSystemReady) { //首次为flase，则不进入该分支
        if (goingCallback != null) {
                goingCallback.run();
            }
        return;
    }
    mLocalDeviceIdleController = LocalServices.getService(DeviceIdleInternal.class);
    mUserController.onSystemReady();
    mAppOpsService.systemReady();
    mProcessList.onSystemReady();
    mSystemReady = true;
}

ArrayList<ProcessRecord> procsToKill = null;
synchronized(mPidsSelfLocked) {
    for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
        ProcessRecord proc = mPidsSelfLocked.valueAt(i);
        //非persistent进程,加入procsToKill
        if (!isAllowedWhileBooting(proc.info)){
            if (procsToKill == null) {
                procsToKill = new ArrayList<ProcessRecord>();
            }
            procsToKill.add(proc);
        }
    }
}
synchronized(this) {
    if (procsToKill != null) {
        //杀掉procsToKill中的进程, 杀掉进程且不允许重启
        for (int i=procsToKill.size()-1; i>=0; i--) {
            ProcessRecord proc = procsToKill.get(i);
            removeProcessLocked(proc, true, false, "system update done");
        }
    }
    mProcessesReady = true; //process处于ready状态
}
Slog.i(TAG, "System now ready");
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY,SystemClock.uptimeMillis());
```

- 启动persistent进程；
- 启动home Activity;
  - 调用 ActivityTaskManagerInternal.startHomeOnAllDisplays ，实际调用ActivityTaskManagerService.startHomeOnAllDisplays()
  - 继续RootWindowContainer.startHomeOnDisplay() -> startHomeOnTaskDisplayArea()，其中startHomeOnTaskDisplayArea()首先通过getHomeIntent()来构建一个category为CATEGORY_HOME的Intent，表明Home Activity；然后通过resolveHomeActivity()从系统所用已安装的引用中找到一个符合HomeItent的Activity，最终调用startHomeActivity()来启动Activity;
  - 最后, mService.getActivityStartController().startHomeActivity()这是调用了者是ActivityStartController.java，ActivityStartController再次调用内部的obtainStarter() 方法返回的是 ActivityStarter 对象，它负责 Activity 的启动，一系列 setXXX() 方法传入启动所需的各种参数；
  - 最终Activity的启动通过执行了ActivityStarter的execute()方法；
- 发送USER_STARTED，USER_STARTING广播；
- 恢复栈顶Activity;
- 发送广播USER_SWITCHED；

```java
synchronized (this) {
    // 启动 persistent 进程
    startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
    mBooting = true;
    // 启动HomeActiviy ，最后通过ActiviityStartController.startHomeActivity()
    mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
    startHomeActivityLocked(mCurrentUserId, "systemReady");
    ...
    long ident = Binder.clearCallingIdentity();
    try {
        //system发送广播USER_STARTED
        Intent intent = new Intent(Intent.ACTION_USER_STARTED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                | Intent.FLAG_RECEIVER_FOREGROUND);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, mCurrentUserId);
        broadcastIntentLocked(...);  

        //system发送广播USER_STARTING
        intent = new Intent(Intent.ACTION_USER_STARTING);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        intent.putExtra(Intent.EXTRA_USER_HANDLE, mCurrentUserId);
        broadcastIntentLocked(...);
    } finally {
        Binder.restoreCallingIdentity(ident);
    }
    mStackSupervisor.resumeTopActivitiesLocked();
    sendUserSwitchBroadcastsLocked(-1, mCurrentUserId);
}

void startPersistentApps(int matchFlags) {
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) return;
    synchronized (this) {
        try {
            final List<ApplicationInfo> apps = AppGlobals.getPackageManager().getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList();
               for (ApplicationInfo app : apps) {
                  if (!"android".equals(app.packageName)) {
                       addAppLocked(app, null, false, null /* ABI override */,
                                ZYGOTE_POLICY_FLAG_BATCH_LAUNCH);
                  }
               }
        } catch (RemoteException ex) {}
    }
}
```

## 四. 总结

1. 创建AMS实例对象，创建Andoid Runtime，ActivityThread, Context对象；
   
   - 创建了3个线程，分别为”ActivityManager”，”android.ui”，”CpuTracker”。
   
   - 创建广播队列（前台广播，后台广播，offload广播）
   
   - 创建/system, /data目录
   
   - 创建ActivityTaskManger

2. setSystemProcess：注册activity、meminfo, cpuinfo等服务到ServiceManager；

3. installSystemProviders，加载SettingsProvider；

4. 启动SystemUIService

5. 调用一系列服务的systemReady()方法, 其中包括AMS的systemReady()；

6. AMS.systemReady主要执行流程
   
   - 杀掉procsToKill中的进程, 杀掉进程且不允许重启;
   - 此时让系统和进程都处于ready状态;
   - 启动persistent进程;
   - 启动home Activity;
     - 调用 ActivityTaskManagerInternal.startHomeOnAllDisplays ，实际内部ActivityTaskManagerService调用该方法；
     - 继续执行RootWindowContainer.startHomeOnDisplay() -> startHomeOnTaskDisplayArea()；
     - 其中startHomeOnTaskDisplayArea()通过getHomeIntent()来构建一个category为CATEGORY_HOME的Intent，表明Home Activity。然后通过resolveHomeActivity()从系统所用已安装的引用中找到一个符合HomeItent的Activity，最终调用startHomeActivity()准备启动Activity;
     - 最后内部通过 ActivityStartController调用了startHomeActivity()，ActivityStartController再通过内部调用obtainStarter() 方法返回ActivityStarter 对象，它负责 Activity 的启动和一系列 setXXX() 方法传入启动所需的各种参数；
     - 最终Activity的启动是通过执行了ActivityStarter的execute()方法完成启动；
   - 发送USER_STARTED，USER_STARTING广播;
   - 恢复栈顶Activity;
   - 发送USER_SWITCHED广播;

### 4.1 发布Binder服务

AMS.setSystemProcess()过程向servicemanager注册了如下这个binder服务

| 服务名         | 类名                     | 功能      |
| ----------- | ---------------------- | ------- |
| activity    | ActivityManagerService | AMS     |
| procstats   | ProcessStatsService    | 进程统计    |
| meminfo     | MemBinder              | 内存      |
| gfxinfo     | GraphicsBinder         | 图像信息    |
| dbinfo      | DbBinder               | 数据库     |
| cpuinfo     | CpuBinder              | CPU     |
| permission  | PermissionController   | 权限      |
| processinfo | ProcessInfoService     | 进程服务    |
| usagestats  | UsageStatsService      | 应用的使用情况 |

想要查看这些服务的信息，可通过`dumpsys <服务名>`命令。比如查看CPU信息命令`dumpsys cpuinfo`。
