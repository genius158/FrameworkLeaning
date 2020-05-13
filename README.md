# FrameworkLeaning
### 编译系统([编译前准备](https://juejin.im/post/5da29dc9f265da5b633cdc8e))
0. [MacOSX10.11.sdk.tar.xz](MacOSX10.11.sdk.tar.xz)解压放到/Applications/XCode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/,前提安装xcode
1. 在 Mac OS 中，可同时打开的文件描述符的默认数量上限太低，在高度并行的编译流程中，可能会超出此上限。要提高此上限，请将下列行添加到 ~/.bash_profile 中：
ulimit -S -n 1024
2. $ source build/envsetup.sh(使用envsetup.sh脚本初始化环境)
3. $ lunch 
- user:设定属性ro.secure=1,打开安全检查功能 \ 设定属性ro.debuggable=0,关闭应用调试功能 \ 默认关闭adb功能 \ 打开Proguard混淆器 \ 打开DEXPREOPT预先编译优化	
- userdebug:设定属性ro.secure=1,打开安全检查功能 \ 设定属性ro.debuggable=1,启用应用调试功能 \ 默认打开adb功能 \ 打开Proguard混淆器 \ 打开DEXPREOPT预先编译优化	
- eng:设定属性ro.secure=0,关闭安全检查功能 \ 设定属性ro.debuggable=1,启用应用调试功能 \ 默认打开adb功能 \ 关闭Proguard混淆器 \ 关闭DEXPREOPT预先编译优化	\ 设定属性ro.kernel.android.checkjni=1,启用JNI调用检查

4. $ sysctl -n machdep.cpu.core_count(查看内核数)
5. $ make -j4（4来自3的数量）
6. emulator 启动模拟器
```
//如果关了终端，想再次启动模拟器执行
source build/envsetup.sh
lunch 5(开始选的编译选项)
emulator
```

### ps 最终编译成功的流程
1. repo sync 可以执行多次保证源码的完整性
2. source build/envsetup.sh
3. lunch
4. make clobber
5. make SELINUX_IGNORE_NEVERALLOWS=true 代替make -j4 用后者试了5、6次都是编译60%多（2、3个小时候后）报错,几乎放弃
- ——常见的一些make编译命令如下所示：
- make 编译工程与模块    //编译整个工程，整体编译时间较长
- make module              //对单个模块进行编译，对其所依赖的模块也进行编译，整体编译时间较长
- mm                              //先进入子目录，对其目录对应的模块进行编译，编译时间短
- mmm                           //编译制定目录下的模块，不编译其所依赖的其他模块，第一次一般都会报错，编译时间短
- make生成镜像文件
- make bootimage        //生成boot.img文件
- make snod                  //重新打包生成system.img文件
- make userdataimage //生成userdata.img文件
- make 无参数               //编译所有的工程

### 生成debug相关文件
1. source build/ensetup.sh  
2. make idegen
3. development/tools/idegen/idegen.sh
- androidStudio打开aosp根目录下生成的android.ipr

## gdb
- source build/envsetup.sh
- lunch 5
1. aosp
- gdbclient.py -p 1234
- set solib-absolute-prefix /Volumes/untitled/WORKING_DIRECTORY/out/target/product/generic_x86/symbols
- set solib-search-path /Volumes/untitled/WORKING_DIRECTORY/out/target/product/generic_x86/symbols
2. normal
- .//prebuilts/gdb/darwin-x86/bin/gdb (根据编译版本而定)
- set solib-absolute-prefix /Volumes/untitled/WORKING_DIRECTORY/out/target/product/generic_x86/symbols
- set solib-search-path /Volumes/untitled/WORKING_DIRECTORY/out/target/product/generic_x86/symbols

## -binder篇（一次注册调用说起）
### 1.非当前进程启动service
Context.bindService(service,conn,flags)->ContextImpl.bindServiceCommon(service,conn,flags,handler,user)
```
//1.
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
    ...
    // 由 ServiceDispatcher 分发最后的绑定回调 InnerConnection.connected
    sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
      
    int res = ActivityManager.getService().bindService(
        mMainThread.getApplicationThread(), getActivityToken(), service,
        service.resolveTypeIfNeeded(getContentResolver()),
        sd, flags, getOpPackageName(), user.getIdentifier());
    ...
}
```
```
//2.
public int bindService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String callingPackage,
            int userId) throws TransactionTooLargeException {
    ...
    synchronized(this) {
        return mServices.bindServiceLocked(caller, token, service,
                resolvedType, connection, flags, callingPackage, userId);
    }
}
```
```
//3.
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {
    ...
    ServiceLookupResult res =
    // 从mServiceMap中查询SerivceRecord缓存，如果没有则创建一个
    retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
        Binder.getCallingUid(), userId, true, callerFg, isBindExternal, allowInstant);

    ServiceRecord s = res.record;
    ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent);
                    
     if (bringUpServiceLocked(s, service.getFlags(), callerFg, false, permissionsReviewRequired) != null) {
      }
}

```
```
//4.
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
    ...
    if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags, hostingType, r.name, false, isolated, false)) == null) {
        } 
        
    // 见19
    requestServiceBindingLocked(s, b.intent, callerFg, false);
    ...
}

```
```
 //4.
private final boolean startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, boolean disableHiddenApiChecks, String abiOverride) {
    // fork进程后，反射生成 ActivityThread
    final String entryPoint = "android.app.ActivityThread";
    
    return startProcessLocked(hostingType, hostingNameStr, entryPoint, app, uid, gids,
        runtimeFlags, mountExternal, seInfo, requiredAbi, instructionSet, invokeWith,
        startTime);
}
```
```
 //6.
  private ProcessStartResult startProcess(String hostingType, String entryPoint,
            ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
            String seInfo, String requiredAbi, String instructionSet, String invokeWith,
            long startTime) {
    startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
        app.info.dataDir, invokeWith,
        new String[] {PROC_START_SEQ_IDENT + app.startSeq});
}
```
```
//7.
public final Process.ProcessStartResult start(final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, int[] gids,
                                                  int runtimeFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, false /* startChildZygote */,
                    zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
        }
    }
```
```
//8.
 private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {

            // 与zygote进程通信
            writer.write(Integer.toString(args.size()));
            writer.newLine();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }

            writer.flush();

            // 等待socket返回结果
            Process.ProcessStartResult result = new Process.ProcessStartResult();

            result.pid = inputStream.readInt();
            if (result.pid < 0) {
            }
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```
```
//9.
Runnable runSelectLoop(String abiList) {
    while (true) {
        ...
        final Runnable command = connection.processOneCommand(this);
        ...
    }
}
```
```
 //10.
 Runnable processOneCommand(ZygoteServer zygoteServer) {
        ...
        
        //com_android_internal_os_Zygote.cpp
        //com_android_internal_os_Zygote_nativeForkAndSpecialize->ForkCommon->fork()
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.runtimeFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.startChildZygote,
                parsedArgs.instructionSet, parsedArgs.appDataDir);
         // 开始新的子进程初始化       
         return handleChildProc(parsedArgs, descriptors, childPipeFd,
                        parsedArgs.startChildZygote);
        ...
    }
```
```
//11.
private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
            FileDescriptor pipeFd, boolean isZygote) {
    ...
    return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
            null /* classLoader */);
    ...
}
```

```
//12.
public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        RuntimeInit.commonInit();
        ZygoteInit.nativeZygoteInit();
        // app 初始化
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }
```
```
//13.
    // applicationInit 方法里调用 findStaticMain
    protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        return new MethodAndArgsCaller(m, argv);
    }
```
```
//14.
static class MethodAndArgsCaller implements Runnable {
        public void run() {
    // 这里将触发 ActivityThread.main 方法
    mMethod.invoke(null, new Object[] { mArgs });
}
```
```
//15.
 // ActivityThread 启动后，回与AMS通信，调用 attachApplication
 @Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        }
    }

```
```
//16.
  private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
    ...
    app.thread.scheduleCreateService(r, r.serviceInfo,
            mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
            app.repProcState);
            
      bringDownServiceLocked(sr);
    
    // 见18
     requestServiceBindingsLocked(r, execInFg);
    ...
}
```
```
//17.
 private void handleCreateService(CreateServiceData data) {
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            service.onCreate();
    }

```
```
//18.
 private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }
```


```
//19.
 private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
    r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
            r.app.repProcState);
}

```
```
//20.
private void handleBindService(BindServiceData data) {
                        IBinder binder = s.onBind(data.intent);
    ...
    ActivityManager.getService().publishService(
         data.token, data.intent, binder);
    ...
}

```
```
//21.
public void publishService(IBinder token, Intent intent, IBinder service) {
    ...
    mServices.publishServiceLocked((ServiceRecord)token, intent, service);
    ...
}
```
```
//22.
 void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    Intent.FilterComparison filter = new Intent.FilterComparison(intent);
    IntentBindRecord b = r.bindings.get(filter);
    if (b != null && !b.received) {
        b.binder = service;
        b.requested = true;
        b.received = true;
        for (int conni=r.connections.size()-1; conni>=0; conni--) {
            ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
            for (int i=0; i<clist.size(); i++) {
                ConnectionRecord c = clist.get(i);
                if (!filter.equals(c.binding.intent.intent)) {
                try {
                    // 这里回调回 最终回调ServiceConnection onServiceConnected
                    c.conn.connected(r.name, service, false);
                } catch (Exception e) {
                }
            }
        }
    }
}

```