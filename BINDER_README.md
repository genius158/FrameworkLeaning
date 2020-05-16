# Binder
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
### 2.service 相关的binder过程
```
//1.
//android.content.ServiceConnection#onServiceConnected
// 从我们最终拿到IBinder往回看
public void onServiceConnected(ComponentName name, IBinder service) {
	mService = IService.Stub.asInterface(service);
}
```
```
//2.
//com.android.server.am.ActiveServices#publishServiceLocked
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    // 注意service参数，也就是onServiceConnected返回的IBinder
    c.conn.connected(r.name, service, false);
}
```

```
//3.
//android.app.ActivityThread#handleBindService
//我们只关心service的binder逻辑，过程中的其他IPC我们忽略
private void handleBindService(BindServiceData data) {
    Service s = mServices.get(data.token);
    IBinder binder = s.onBind(data.intent);
    
    // 见6
    ActivityManager.getService().publishService(
            data.token, data.intent, binder);
}

```

```
//4.
//com.yan.text.TestIntentService
//见5
class MyBinder extends IMyAidlInterface.Stub{
        @Override
        public String getName() throws RemoteException{
            return "test";
        }
    }
    MyBinder mBinder = MyBinder();
	@Override
	public IBinder onBind(Intent intent) {
	    // 我们自己实现的binder对象
		return mBinder;
	}

```
```
//5.
//com.yan.text.IMyAidlInterface
public interface IMyAidlInterface extends android.os.IInterface {
  public static abstract class Stub extends android.os.Binder
      implements com.yan.text.IMyAidlInterface {
    private static final java.lang.String DESCRIPTOR = "com.yan.text.IMyAidlInterface";

    public Stub() {
      this.attachInterface(this, DESCRIPTOR);
    }

    /**
     * client端回调
     */
    public static com.yan.text.IMyAidlInterface asInterface(android.os.IBinder obj) {
      if ((obj == null)) {
        return null;
      }
      // 本进程下找到对应的binder，则直接使用该对象
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin != null) && (iin instanceof com.yan.text.IMyAidlInterface))) {
        return ((com.yan.text.IMyAidlInterface) iin);
      }
      // 否则生成服务binder的代理对象
      return new com.yan.text.IMyAidlInterface.Stub.Proxy(obj);
    }

    @Override public android.os.IBinder asBinder() {
      return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
        throws android.os.RemoteException {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code) {
        case INTERFACE_TRANSACTION: {
          reply.writeString(descriptor);
          return true;
        }
        // getServiceName方法，Stub作为服务端，这里接受到客户端的getServiceName请求，返回数据
        case TRANSACTION_getServiceName: {
          // Write RPC headers.  (previously just the interface token) 来自writeInterfaceToken注释
          // 校验 与客户端的writeInterfaceToken()写入的descriptor比较是否相等
          data.enforceInterface(descriptor);
          
          java.lang.String _result = this.getServiceName();
          
          // reply 即Parcel 容器，对应frameworks/native/libs/binder/Parcel.cpp
          // 可以包含数据或者是对象引用，主要用于Binder的传输
          // 同时支持序列化以及跨进程之后进行反序列化
          
          // 把数据装入容器
          reply.writeString(_result);
          return true;
        }
        default: {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }

    private static class Proxy implements com.yan.text.IMyAidlInterface {
      private android.os.IBinder mRemote;

      // 远程binder引用
      Proxy(android.os.IBinder remote) {
        mRemote = remote;
      }

      @Override public android.os.IBinder asBinder() {
        return mRemote;
      }

      public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
      }

      @Override public java.lang.String getServiceName() throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.lang.String _result;
        try {
          // 对应 服务端enforceInterface
          _data.writeInterfaceToken(DESCRIPTOR);
          
          // mRemote 即native 的BPBinder
          mRemote.transact(Stub.TRANSACTION_getServiceName, _data, _reply, 0);
          
          // 得到远程服务端返回的数据
          _result = _reply.readString();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
    }

    static final int TRANSACTION_getServiceName = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
  }

  public java.lang.String getServiceName() throws android.os.RemoteException;
}

```
```
//6. android8以上IActivityManager.Stub代码被隐藏了，以下伪代码（android7.0）
public void publishService(IBinder token, Intent intent, IBinder service) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(token);
    intent.writeToParcel(data, 0);
    //将service.onBind的返回值传递给远程进程
    data.writeStrongBinder(service);
    mRemote.transact(PUBLISH_SERVICE_TRANSACTION, data, reply, 0);
    reply.readException();
    data.recycle();
    reply.recycle();
}
```
```
//7.
//android.os.BinderProxy#transact
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    ...
    return transactNative(code, data, reply, flags);
    ...
}
```
```
//8.
//frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    // java parcel 转c++
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);

    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    target->transact(code, *data, reply, flags); 
}
```

```
//9.
//frameworks/base/core/jni/android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);

    //见10
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    //见11
    target->transact(code, *data, reply, flags); 
}
```
```
//10.
//frameworks/base/core/jni/android_util_Binder.cpp
BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
    /**
     * C++ pointer to BinderProxyNativeData. That consists of strong pointers to the
     * native IBinder object, and a DeathRecipientList.
     */
    //private final long mNativeData;
    //gBinderProxyOffsets.mNativeData 对应BinderProxy mNativeData(对应native BPBinder指针)
    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
}
```

```
//11.
///frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    ...
    status_t status = IPCThreadState::self()->transact(
        mHandle, code, data, reply, flags);
    ...    
}
```
```
//12.
//frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState* IPCThreadState::self()
{
    const pthread_key_t k = gTLS;
    // 尝试从当前线程中获取出IPCThreadState指针
    IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
    if (st) return st;
    // 不存在新建IPCThreadState
    return new IPCThreadState;
}
```
```
//13.
//frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    //组装binder_transaction_data结构体
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    //TF_ONE_WAY代表是单向通信，不需要回复
    //(flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY"
    if ((flags & TF_ONE_WAY) == 0) {
           if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(nullptr, nullptr);
    }
}
```

```
//14.
//frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;
    tr.target.ptr = 0; //binder_node的指针
    tr.target.handle = handle;//用它可以找到对应的binder_ref，通过它找到目标的binder_node，最后找到目标进程binder_proc
    tr.code = code;//Client端与Server端双方约定的命令码,这里是PUBLISH_SERVICE_TRANSACTION
    tr.flags = binderFlags;
    tr.cookie = 0;//BBinder指针
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    tr.data_size = data.ipcDataSize();
    tr.data.ptr.buffer = data.ipcData();
    tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
    tr.data.ptr.offsets = data.ipcObjects();

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```
```
//15.
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;
    while (1) {
        //与Binder驱动进行数据收发的函数
        if ((err=talkWithDriver()) < NO_ERROR) break; 
        err = mIn.errorCheck();
        if (mIn.dataAvail() == 0) continue;
        cmd = (uint32_t)mIn.readInt32();
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;break;
        case BR_DEAD_REPLY:
            err = DEAD_OBJECT; goto finish;
        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION; goto finish;
        case BR_ACQUIRE_RESULT:
            {
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;
        
        //解析来自服务器端的响应
        case BR_REPLY: 
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                if (err != NO_ERROR) goto finish;
                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            //解析 命令BR_TRANSACTION系列
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }
    return err;
}
```
```
//16.

status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD <= 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // 是否需要读取数据
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
 
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
    ////需要发送的数据长度
    bwr.write_size = outAvail;
    //需要发送的数据的起始地址赋值到bwr.write_buffer
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        //回应的结果数据的大小
        bwr.read_size = mIn.dataCapacity();
        //数据bwr.read_buffer即mIn.data()开始的地址 
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        // 通过linux命令ioctl与binder驱动进行通信
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
    } while (err == -EINTR);
 }
 
```
```
//17.
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    case BR_ERROR:
        result = mIn.readInt32(); break;
    case BR_OK:
        break;
    case BR_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();       
        obj->incStrong(mProcess.get());       
        mOut.writeInt32(BC_ACQUIRE_DONE);
        mOut.writePointer((uintptr_t)refs);
        mOut.writePointer((uintptr_t)obj);
        break;

    case BR_RELEASE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();      
        mPendingStrongDerefs.push(obj);
        break;

    case BR_INCREFS:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        refs->incWeak(mProcess.get());
        mOut.writeInt32(BC_INCREFS_DONE);
        mOut.writePointer((uintptr_t)refs);
        mOut.writePointer((uintptr_t)obj);
        break;

    case BR_DECREFS:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        mPendingWeakDerefs.push(refs);
        break;

    case BR_ATTEMPT_ACQUIRE:
        refs = (RefBase::weakref_type*)mIn.readPointer();
        obj = (BBinder*)mIn.readPointer();
        {
            const bool success = refs->attemptIncStrong(mProcess.get());
            mOut.writeInt32(BC_ACQUIRE_RESULT);
            mOut.writeInt32((int32_t)success);
        }
        break;
    //驱动回应的BR_TRANSACTION
    case BR_TRANSACTION: 
        {
            binder_transaction_data tr;
            // 读取数据到 tr
            result = mIn.read(&tr, sizeof(tr));           
            if (result != NO_ERROR) break;
            Parcel buffer;
            //绑定释放
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;

            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;          

            Parcel reply;
            status_t error;          
            if (tr.target.ptr) {
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    // 由BBinder transact 调用到 其子类JavaBBinder::transact 
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }
            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }           

            if ((tr.flags & TF_ONE_WAY) == 0) {
                LOG_ONEWAY("Sending reply to %d!", mCallingPid);
                if (error < NO_ERROR) reply.setError(error);
                //回传调用者数据
                sendReply(reply, 0);
            } else {
                LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
            }

            mCallingPid = origPid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;   
        }
        break;  
    return result;
}
```
```
//18. 
//JavaBBinder
virtual status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0)
    {
        JNIEnv* env = javavm_to_jnienv(mVM);
        IPCThreadState* thread_state = IPCThreadState::self();
        
        调用java层 binder对象的ExecTransact 进而调用子类的onTransact
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);

    }
```
```
//19.
// drivers/staging/android/binder.c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	// 从file结构体的private_data字段取出当前进程的binder_proc
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;

	thread = binder_get_thread(proc);
	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		break;
	case BINDER_SET_MAX_THREADS:
		if (copy_from_user(&proc->max_threads, ubuf, sizeof(proc->max_threads))) {
		}
		break;
	case BINDER_SET_CONTEXT_MGR:
		ret = binder_ioctl_set_ctx_mgr(filp);
		if (ret)
			goto err;
		ret = security_binder_set_context_mgr(proc->tsk);
		if (ret < 0)
			goto err;
		break;
	case BINDER_THREAD_EXIT:
		binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
			     proc->pid, thread->pid);
		binder_free_thread(proc, thread);
		thread = NULL;
		break;
	case BINDER_VERSION: {
		struct binder_version __user *ver = ubuf;

		if (size != sizeof(struct binder_version)) {
			ret = -EINVAL;
			goto err;
		}
		if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
			     &ver->protocol_version)) {
			ret = -EINVAL;
			goto err;
		}
		break;
	}
	default:
		ret = -EINVAL;
		goto err;
	}
	ret = 0;
err:
	if (thread)
		thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
	binder_unlock(__func__);
	wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	if (ret && ret != -ERESTARTSYS)
		pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
	trace_binder_ioctl_done(ret);
	return ret;
}

```
```
//20.
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;

    // 从用户空间的bwr拷贝到内核空间的binder_write_read
	if (copy_from_user(&bwr, ubuf, sizeof(bwr))) {
	}

    // binder_write_read存在写数据则先执行binder_thread_write
	if (bwr.write_size > 0) {
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
	}
	if (bwr.read_size > 0) {
	    //读过程有可能会使进程进入休眠
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
	    // 当前进程todo队列有数据，唤醒等待进程
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait);
		}
	}
}

```
```
//21.
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr < end && thread->return_error == BR_OK) {
		if (get_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		
		switch (cmd) {
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
            // 拷贝用户空间的binder_transaction_data
			if (copy_from_user(&tr, ptr, sizeof(tr)))
				return -EFAULT;
			ptr += sizeof(tr);
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
			break;
		}
    }
}

```
```
22.
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
    struct binder_transaction *t;
	if (reply) {
	} else {
	    // handle为0代表serverManger
		if (tr->target.handle) {
			struct binder_ref *ref;
			//见23
            //从当前进程proc获取binder_ref(当前进程使用到来自别的进程的binder服务的引用，与binder_node 对应)
			ref = binder_get_ref(proc, tr->target.handle);
			target_node = ref->node;
		}
		 //target_proc为AMS所在进程system_process
		target_proc = target_node->proc;
    }
    // system_process存在空闲线程，使用线程的todo list否则是否进程的
	if (target_thread) {
		e->to_thread = target_thread->pid;
		target_list = &target_thread->todo;
		target_wait = &target_thread->wait;
	} else {
		target_list = &target_proc->todo;
		target_wait = &target_proc->wait;
	}
	
    //非oneway的通信方式，把当前thread保存到transaction的from字段
    if (!reply && !(tr->flags & TF_ONE_WAY))
        t->from = thread;
    else
        t->from = NULL;

    t->sender_euid = task_euid(proc->tsk);
    t->to_proc = target_proc; //system_process
    t->to_thread = target_thread;
    t->code = tr->code;  //此次通信code = PUBLISH_SERVICE_TRANSACTION

    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

    t->buffer->allow_user_free = 0;
    t->buffer->transaction = t;
    t->buffer->target_node = target_node;

    if (target_node)
        //引用计数加1
        binder_inc_node(target_node, 1, 0, NULL); 

    offp = (binder_size_t *)(t->buffer->data + ALIGN(tr->data_size, sizeof(void *)));

    //分别拷贝用户空间的binder_transaction_data中ptr.buffer和ptr.offsets到内核
    copy_from_user(t->buffer->data,
        (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size);

    off_end = (void *)offp + tr->offsets_size;

	for (; offp < off_end; offp++) {
		struct flat_binder_object *fp;
		fp = (struct flat_binder_object *)(t->buffer->data + *offp);
		off_min = *offp + sizeof(struct flat_binder_object);
		switch (fp->type) {

        // BINDER_TYPE_BINDER和BINDER_TYPE_WEAK_BINDER类型的flat_binder_object传输发生在：
        // Server进程主动向Client进程发送Service (匿名Service )
		// Server进程向Service Manager进程注册Service
        case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct binder_ref *ref;
			struct binder_node *node = binder_get_node(proc, fp->binder);

			if (node == NULL) {
			    //当前进程 创建binder_node
				node = binder_new_node(proc, fp->binder, fp->cookie);
			}
			if (fp->cookie != node->cookie) {
			}
            // 在目标进程，寻找binder_ref关联当前进程的bind_node
			ref = binder_get_ref_for_node(target_proc, node);
			if (fp->type == BINDER_TYPE_BINDER)
				fp->type = BINDER_TYPE_HANDLE;
			else
				fp->type = BINDER_TYPE_WEAK_HANDLE;
			fp->handle = ref->desc;
			binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
				       &thread->todo);
		} break;

        // BINDER_TYPE_HANDLE和BINDER_TYPE_WEAK_HANDLE类型的flat_binder_object传输发生在：
        // 一个Client向另外一个进程发送Service代理
		case BINDER_TYPE_HANDLE:
		case BINDER_TYPE_WEAK_HANDLE: {
			struct binder_ref *ref = binder_get_ref(proc, fp->handle);

			if (ref->node->proc == target_proc) {
				if (fp->type == BINDER_TYPE_HANDLE)
					fp->type = BINDER_TYPE_BINDER;
				else
					fp->type = BINDER_TYPE_WEAK_BINDER;
				fp->binder = ref->node->ptr;
				fp->cookie = ref->node->cookie;
				binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
			} else {
				struct binder_ref *new_ref;
				new_ref = binder_get_ref_for_node(target_proc, ref->node);
				fp->handle = new_ref->desc;
				binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
			}
		} break;

		default:
			return_error = BR_FAILED_REPLY;
			goto err_bad_object_type;
		}
	}
	if (reply) {
		binder_pop_transaction(target_thread, in_reply_to);
	} else if (!(t->flags & TF_ONE_WAY)) {
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
	} else {
    // 需要回复的ipc,拿出目标的todo 队列
	if (target_node->has_async_transaction) {
			target_list = &target_node->async_todo;
			target_wait = NULL;
		} else
			target_node->has_async_transaction = 1;
	}
	//添加事物到目标队列
	t->work.type = BINDER_WORK_TRANSACTION;
	list_add_tail(&t->work.entry, target_list);
	
	tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	list_add_tail(&tcomplete->entry, &thread->todo);
	if (target_wait)
	    // 唤醒目标等待队列
		wake_up_interruptible(target_wait);
	return;
}
```
```
23.
static struct binder_ref *binder_get_ref(struct binder_proc *proc,
					 uint32_t desc)
{
    //从当前进程proc中的refs_by_desc获取binder_ref 也就是AMS在在client的引用
	struct rb_node *n = proc->refs_by_desc.rb_node;
	struct binder_ref *ref;
    // 遍历红黑树，匹配对应的binder_ref
    // binder_ref是在获取AMS服务的时候，binder驱动加到client的
	while (n) {
		ref = rb_entry(n, struct binder_ref, rb_node_desc);

		if (desc < ref->desc)
			n = n->rb_left;
		else if (desc > ref->desc)
			n = n->rb_right;
		else
			return ref;
	}
	return NULL;
}
```
```
//24.
static struct binder_ref *binder_get_ref_for_node(struct binder_proc *proc,
						  struct binder_node *node)
{
	struct rb_node *n;
	struct rb_node **p = &proc->refs_by_node.rb_node;
	struct rb_node *parent = NULL;
	struct binder_ref *ref, *new_ref;

    // 尝试从目标进程binder_proc中找到binder_ref
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_node);

		if (node < ref->node)
			p = &(*p)->rb_left;
		else if (node > ref->node)
			p = &(*p)->rb_right;
		else
			return ref;
	}
    // 没有找到，则创建对象
	new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
	if (new_ref == NULL)
		return NULL;
	binder_stats_created(BINDER_STAT_REF);
	new_ref->debug_id = ++binder_last_id;
	new_ref->proc = proc;
	new_ref->node = node;
	rb_link_node(&new_ref->rb_node_node, parent, p);
	rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);

    // 生成引用号，如果是serverManager则固定是0
	new_ref->desc = (node == binder_context_mgr_node) ? 0 : 1;
	for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
		ref = rb_entry(n, struct binder_ref, rb_node_desc);
		if (ref->desc > new_ref->desc)
			break;
		new_ref->desc = ref->desc + 1;
	}

    //找到插入点
	p = &proc->refs_by_desc.rb_node;
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_desc);

		if (new_ref->desc < ref->desc)
			p = &(*p)->rb_left;
		else if (new_ref->desc > ref->desc)
			p = &(*p)->rb_right;
		else
			BUG();
	}
    // 把binder_ref根据desc 插入到目标进程proc的refs_by_desc
    rb_link_node(&new_ref->rb_node_desc, parent, p);
	rb_insert_color(&new_ref->rb_node_desc, &proc->refs_by_desc);
	if (node) {
		hlist_add_head(&new_ref->node_entry, &node->refs);
	}
	return new_ref;
}

```
```
//25
static int binder_thread_read(struct binder_proc *proc,
                  struct binder_thread *thread,
                  void  __user *buffer, int size,
                  signed long *consumed, int non_block)
{
    void __user *ptr = buffer + *consumed;
    void __user *end = buffer + size;

    int ret = 0;
    int wait_for_proc_work;

    if (*consumed == 0) {
        if (put_user(BR_NOOP, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
    }

retry:
    wait_for_proc_work = thread->transaction_stack == NULL &&
                list_empty(&thread->todo);
    thread->looper |= BINDER_LOOPER_STATE_WAITING;
    if (wait_for_proc_work)
        proc->ready_threads++;
    if (wait_for_proc_work) {
            ret = wait_event_freezable_exclusive(proc->wait, binder_has_proc_work(proc, thread));
    } else {
            ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    }
    if (wait_for_proc_work)
        proc->ready_threads--;
    thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

    if (ret)
        return ret;

    while (1) {
        uint32_t cmd;
        struct binder_transaction_data tr;
        struct binder_work *w;
        struct binder_transaction *t = NULL;
        // 读取todo列表首节点，这正是在Client端触发的binder_transaction
        // 在结尾处插进来的t->work.entry
        if (!list_empty(&thread->todo))
            w = list_first_entry(&thread->todo, struct binder_work, entry);
        else if (!list_empty(&proc->todo) && wait_for_proc_work)
            w = list_first_entry(&proc->todo, struct binder_work, entry);
        ... ...
        switch (w->type) {
        case BINDER_WORK_TRANSACTION: {
            // 根据binder_work找到他所在的binder_transaction
            t = container_of(w, struct binder_transaction, work);
        } break;
        ... ...
        //组装一个binder_transaction_data结构体，从发起端的binder_transaction以及binder_buffer中提取数据
        if (t->buffer->target_node) {
            struct binder_node *target_node = t->buffer->target_node;
            tr.target.ptr = target_node->ptr;
            tr.cookie =  target_node->cookie;
            t->saved_priority = task_nice(current);
            if (t->priority < target_node->min_priority &&
                !(t->flags & TF_ONE_WAY))
                binder_set_nice(t->priority);
            else if (!(t->flags & TF_ONE_WAY) ||
                 t->saved_priority > target_node->min_priority)
                binder_set_nice(target_node->min_priority);
            cmd = BR_TRANSACTION;
        } ... ...
        tr.code = t->code;//
        tr.flags = t->flags;
        tr.sender_euid = t->sender_euid;

        if (t->from) {
            struct task_struct *sender = t->from->proc->tsk;
            tr.sender_pid = task_tgid_nr_ns(sender,
                            current->nsproxy->pid_ns);
        } else {
            tr.sender_pid = 0;
        }

        tr.data_size = t->buffer->data_size;
        tr.offsets_size = t->buffer->offsets_size;
        tr.data.ptr.buffer = (void *)t->buffer->data +
                    proc->user_buffer_offset;
        tr.data.ptr.offsets = tr.data.ptr.buffer +
                    ALIGN(t->buffer->data_size,
                        sizeof(void *));

        if (put_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
        ptr += sizeof(uint32_t);
        // 将binder_transaction_data结构体拷贝到用户空间，由AMS接收
        if (copy_to_user(ptr, &tr, sizeof(tr))) 
            return -EFAULT;
        ptr += sizeof(tr);
        // 执行后删除这次todo
        list_del(&t->work.entry);
        t->buffer->allow_user_free = 1;
    }

    return 0;
}
```
```
//26.
//由18 数据被返回到AMS的BBinder,伪代码
 public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
              throws RemoteException {
    case PUBLISH_SERVICE_TRANSACTION: {
        data.enforceInterface(IActivityManager.descriptor);
        // readStrongBinder将返回BBinder的代理
        IBinder token = data.readStrongBinder();
        Intent intent = Intent.CREATOR.createFromParcel(data);
        IBinder service = data.readStrongBinder();
        // 这里回到第一大点（1.非当前进程启动service）的第21步，从而使client端最后回调到onServiceConnected
        // 返回服务端的binder代理
        publishService(token, intent, service);
        reply.writeNoException();
        return true;
    }
}

```
```
//27.
//native/libs/binder/Parcel.cpp
// readStrongBinder 最终调到
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);

    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER:
                //同一进程，从flat中的cookie也就是BBinder的指针返回binder对象
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                // 非同一进程，返回BpBinder
                *out = proc->getStrongProxyForHandle(flat->handle);
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```
```
//28.
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    handle_entry* e = lookupHandleLocked(handle);
    if (e != nullptr) {
        IBinder* b = e->binder;
        //新创建的binder 一定为空
        if (b == nullptr || !e->refs->attemptIncWeak(this)) {
            //创建一个BpBinder对象
            b = BpBinder::create(handle);
            //放入handle_entry，缓存，方便后面查询
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```
