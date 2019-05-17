# HandlerThread原理分析

HandlerThread是一个内部拥有Handler和Looper的特殊Thread，可以方便地在子线程中处理消息。

## 简单使用

HandlerThread的使用比较简单。

```java
    mHandlerThread = new HandlerThread(THREAD_NAME);
    mHandlerThread.start();
```

首先,实例化一个HandlerThread，然后调用start()方法。在start()方法中，会调用run()方法：

```java
    @Override
    public void run() {
        mTid = Process.myTid();
		//实例化looper对象
        Looper.prepare();
        synchronized (this) {
			//获取looper对象
            mLooper = Looper.myLooper();
			//通知其它线程
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
		//开启循环
        Looper.loop();
        mTid = -1;
    }
```

然后，定义处理子线程消息的Handler:

```java
    mThreadLooper = mHandlerThread.getLooper();
	
	mThreadHandler = new Handler(mThreadLooper, new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_THREAD_UPDATE:
                        //在子线程中执行耗时任务
                        SystemClock.sleep(3000);
                        mMainHandler.sendEmptyMessage(MSG_MAIN_UPDATE);
                        break;
                    default:
                        break;
                }
                return false;
            }
        });
```

在HandlerThread.getLooper()方法中:

```java
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }
```

在getLooper()方法中，由于子线程可能还没有准备好looper,因此，会调用wait()方法等待，如果子线程looper已经准备好了，则会通过notifyAll()来唤醒。

在子线程中可以执行耗时的操作，执行完成后，可以通过在UI线程的Handler发送消息去通知UI变更。

```java
    mMainHandler.sendEmptyMessage(MSG_MAIN_UPDATE);
```

UI线程的Handler:

```java
    static class MainHandler extends Handler {

		//为防止内存泄漏，引入WeakReference
        private WeakReference<Activity> mWeakReference;

        public MainHandler(Activity activity) {
            mWeakReference = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) mWeakReference.get();
            if (activity != null) {
                switch (msg.what) {
                    case MSG_MAIN_UPDATE:
                        activity.updateInfo();
                        break;
                    default:
                        break;
                }
            }

        }
    }
```

为防止内存泄漏，引入WeakReference。在onDestory()方法中，移除所有消息:

```java
    mMainHandler.removeCallbacksAndMessages(null);
	
	mThreadLooper.quit();
```

源码地址:[https://github.com/milovetingting/Samples/tree/master/HandlerThread](https://github.com/milovetingting/Samples/tree/master/HandlerThread)
