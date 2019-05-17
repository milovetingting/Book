# IntentService原理分析

IntentService是一个异步处理请求的服务，通过Context#startService(Intent)可以将请求发送给IntentService,IntentService在工作线程中依次串行处理每一个Intent，当处理完所有请求后，IntentService会自动停止。

在IntentService内部是通过HandlerThread来切换线程和处理消息的。

当IntentService首次启动时,会调用onCreate()方法:

```java
    @Override
    public void onCreate() {
        super.onCreate();
		//创建HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        //启动线程
		thread.start();

		//从HandlerThread中获取looper
        mServiceLooper = thread.getLooper();
		//实例化Handler
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

在onCreate()方法中，首先创建了HandlerThread，然后启动它。然后从创建的thread中获取looper，并实例化Handler。

onStartCommand()方法：

```java
    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
```

onStartCommand()方法，在自定义类继承自IntentService时，不要去重写，而应该重写onHandleIntent()方法。在onStartCommand()方法中，调用了onStart()方法，将intent传入Message，并发送出去。

onStart()方法:

```java
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```

在onStart()方法中，将intent信息打包到Message中，并发送到消息队列。

Message消息的处理:

```java
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

onStart()方法将intent传入了Message,最终在ServiceHandler的handleMessage()方法处理，在handleMessage()方法中，回调了onHandleIntent()方法，这个方法是需要我们重写的,由于ServiceHandler是运行在子线程中，所以onHandleIntent()的执行也会在子线程中。当依次执行完任务后，调用了stopSelf(startId)方法停止Service。

stopSelf(startId)方法，只有当startId和最后启动Service时的startId一致时，才会停止服务,所以如果还有任务没有执行完成，则不会成功停止服务。

```java
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }
```

在onDestory()方法中，调用Looper的quit()方法，退出消息循环。