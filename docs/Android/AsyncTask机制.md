# AsyncTask机制

AsyncTask可以让我们更容易地使用UI线程。它允许执行后台操作，并把结果发布到UI线程上，而不需要操作线程或Handler。AsyncTask被设计成一个和Thread、Handler相关的一个帮助类。AsyncTask用于短时(最多是几秒)的操作。

AsyncTask使用需要注意以下几点:

> - AsyncTask类必须在UI线程上加载。AsyncTask必须在UI线程实例化。execute()方法也必须在UI线程调用。
> 
> - 不要手动调用onPreExecute()、onPostExecute()、doInBackground()、onProgressUpdate()方法。
> 
> - 每个AsyncTask实例只能调用一次execute，如果再次调用，则会抛出异常。

**AsyncTask首次引入时，AsyncTask中的任务是串行的。从Android1.6之后，AsyncTask被设计成并行的。从Android3.0后，AsyncTask被重新设计成串行。如果在3.0后的版本需要并行，则可以调用AsyncTask的executeOnExecutor(java.util.concurrent.Executor, Object[])方法，手动传入Executor。**

在AsyncTask类加载时，会初始化ThreadPoolExecutor:

```java
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

其中，核心线程数,最小为2个，最大为4个:

```java
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
```

最大线程数CPU数量*2+1:

```java
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
```

KeepAlive时间为30s:

```java
    private static final int KEEP_ALIVE_SECONDS = 30;
```

任务队列最大是128：

```java
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
```


AsyncTask的基本使用:

1、定义一个类，继承自AsyncTask，根据需要重写doInBackground()、onProgressUpdate()、onPostExecute()方法，一般doInBackground()、onPostExecute()方法是需要重写的，在这里实现自己的业务。doInBackground()方法运行在子线程中。onProgressUpdate()和onPostExecute()运行在UI线程。

```java    
	private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
	      protected Long doInBackground(URL... urls) {
	          int count = urls.length;
	          long totalSize = 0;
	          for (int i = 0; i < count; i++) {
	              totalSize += Downloader.downloadFile(urls[i]);
	              publishProgress((int) ((i / (float) count) * 100));
	              // Escape early if cancel() is called
	              if (isCancelled()) break;
	          }
	          return totalSize;
	      }
	 
	      protected void onProgressUpdate(Integer... progress) {
	          setProgressPercent(progress[0]);
	      }
	 
	      protected void onPostExecute(Long result) {
	          showDialog("Downloaded " + result + " bytes");
	      }
	  }
```

2、创建DownloadFilesTask的实例，并执行execute()方法:

```java
    new DownloadFilesTask().execute(url1, url2, url3);
```

下面，从源码角度来分析下AsyncTask的原理。

AsyncTask的执行入口是execute方法：

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

execute()方法必须在UI线程调用。在方法内部调用了executeOnExecutor()方法。

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
		//检查AsyncTask状态，不是未执行状态(如任务正在运行或已完成)，则会抛出相应异常
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

		//将状态置为RUNNING
        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

executeOnExecutor()方法也必须在UI线程调用。在方法开始时，会检查AsyncTask状态，不是未执行状态(如任务正在运行或已完成)，则会抛出相应异常。然后，将任务状态置为RUNNING状态,调用onPreExecute()方法，这个方法需要自己重写，可以做一些UI提示。然后，将参数设置为mWorker，调用Executor的execute()方法。

如果使用默认的Executor，则为串行。

```java
    @MainThread
    public static void execute(Runnable runnable) {
        sDefaultExecutor.execute(runnable);
    }
```

接下来，看看sDefaultExecutor的定义:

```java
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

而SERIAL_EXECUTOR的具体实现如下:

```java
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

	private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

mWorker的定义:

```java
    mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
					//将线程设置为后台线程
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
					//调用doInBackground方法
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
					//发送结果
                    postResult(result);
                }
                return result;
            }
        };
```

当执行execute()方法，会调用mWorker的call()方法，在此方法中，会将线程设置为后台线程，然后调用doInBackground()方法，并在执行完成后调用postResult()方法。在doInBackground()方法中，可以调用publishProgress()方法，将进度信息发送到UI线程中。

postResult()方法:

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

发送一个Message到Handler中.

```java
    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```

在Handler的handleMessage()方法中处理消息。如果已经执行完成，则会调用AsyncTask的finish()方法,如果是更新进度，则会调用AsyncTask的onProgressUpdate()方法：

```java
    private void finish(Result result) {
        if (isCancelled()) {
			//如果是取消任务，则回调onCancelled()方法。
            onCancelled(result);
        } else {
			//回调onPostExecute()方法
            onPostExecute(result);
        }
		//设置状态为FINISHED
        mStatus = Status.FINISHED;
    }

	@MainThread
    protected void onProgressUpdate(Progress... values) {
    }

publishProgress()方法:

    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```