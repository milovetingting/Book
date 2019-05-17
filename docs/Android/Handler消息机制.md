# Handler消息机制

Handler消息机制主要涉及Looper、Handler、MessageQueue、Message。其中，Looper主要负责获取消息，Handler负责发送消息及处理消息，MessageQueue是消息队列，Message是消息类。

## Looper循环获取消息

> 1、ActivityThread的main()方法:

```java
    public static void main(String[] args) {

        ...

		//准备looper
        Looper.prepareMainLooper();

        ...

		//进入无限循环
        Looper.loop();

		//如果loop()循环退出，则抛出异常，整个应用退出
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

> 2、prepareMainLooper()方法:
```java
    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
		//初始化looper
        prepare(false);
        synchronized (Looper.class) {
			//如果已经设置过sMainLooper，则抛出异常。每个线程中只允许存在一个looper。
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
			//设置sMainLooper
            sMainLooper = myLooper();
        }
    }
```

> 3、在prepareMainLooper()方法中，首先调用prepare(false)方法:

```java
    private static void prepare(boolean quitAllowed) {
		//如果ThreadLocal中已经存在looper,则抛出异常
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
		//如果没有初始化looper，则将looper保存到ThradLocal中。
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

> 4、在prepare()方法中调用Looper的构造方法初始化MessageQueue:

```java
    private Looper(boolean quitAllowed) {
		//初始化MessageQueue
        mQueue = new MessageQueue(quitAllowed);
		//设置当前线程给mThread变量
        mThread = Thread.currentThread();
    }
```

> 5、在prepareMainLooper()方法调用prepare(false)方法后，会调用myLooper()方法:

```java
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
		//将保存在ThreadLocal中的looper返回
        return sThreadLocal.get();
    }
```

> 5、到这里，prepareMainLooper()方法执行完毕。然后执行Looper.loop()方法:

```java
	/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {

		//获取looper
        final Looper me = myLooper();
		//如果looper为null,则抛出异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
		//获取MessageQueue
        final MessageQueue queue = me.mQueue;

		...

		//开启无限循环
        for (;;) {
			//从消息队列中取消息，如果没有消息，则会阻塞
            Message msg = queue.next(); // might block
			//如果消息为null，则表示退出循环
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
			
			...
          
            try {
				//回调target，即Handler的dispatchMessage方法
                msg.target.dispatchMessage(msg);
                ...
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            
			...

            msg.recycleUnchecked();
        }
    }
```

> 首先，获取looper,如果没有设置过looper，则抛出异常。然后，开启无限循环，通过looper的MessageQueue，不停获取消息，如果没有消息，则阻塞。如果获取到了消息，则会回调Handler的dispatchMessage方法，方法执行会切换到Handler的线程。

> 6、MessageQueue的next()方法:

```java
    Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```

> 在next()方法中，通过for(;;)开启无限循环去获取消息，如果获取到消息则返回。

## Handler发送消息

> 1、sendMessage()方法:

```java
    /**
     * Pushes a message onto the end of the message queue after all pending messages
     * before the current time. It will be received in {@link #handleMessage},
     * in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     */
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

> 2、sendMessage()方法会调用sendMessageDelayed()方法:

```java
    /**
     * Enqueue a message into the message queue after all pending messages
     * before (current time + delayMillis). You will receive it in
     * {@link #handleMessage}, in the thread attached to this handler.
     *  
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

> 3、sendMessageDelayed()方法会调用sendMessageAtTime():

```java
    /**
     * Enqueue a message into the message queue after all pending messages
     * before the absolute time (in milliseconds) <var>uptimeMillis</var>.
     * <b>The time-base is {@link android.os.SystemClock#uptimeMillis}.</b>
     * Time spent in deep sleep will add an additional delay to execution.
     * You will receive it in {@link #handleMessage}, in the thread attached
     * to this handler.
     * 
     * @param uptimeMillis The absolute time at which the message should be
     *         delivered, using the
     *         {@link android.os.SystemClock#uptimeMillis} time-base.
     *         
     * @return Returns true if the message was successfully placed in to the 
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.  Note that a
     *         result of true does not mean the message will be processed -- if
     *         the looper is quit before the delivery time of the message
     *         occurs then the message will be dropped.
     */
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

> 4、在这个方法中，会通过MessageQueue的enqueueMessage()方法，将消息发送到消息队列中。

> 5、enqueueMessage()方法:

```java
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
		//设置Message的target为当前的Handler，以便获取到消息后能回调dispatchMessage方法。
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

> 6、MessageQueue的enqueueMessage()方法:

```java
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

> sendMessageAtTime()方法中的mQueue是在Handler的构造方法中赋值的:

```java
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
			//检测是否会有泄漏
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

		//获取looper
        mLooper = Looper.myLooper();
		//如果looper为null,则抛出异常
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
		//设置mQueue
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

> 如果没有传入looper，则会通过Looper.myLooper()获取looper,如果没有在线程中设置过looper，则会抛出异常

```java
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

> 如果传入了looper，则直接设置mQueue。

## Handler处理消息

> dispatchMessage()方法,如果Message设置了callBack,则会回调callBack的run()方法；如果Message没有设置callBack,在这种情况下，如果Handler的callBack不为null，则会回调handleMessage()方法;如果Handler没有设置callBack或者Handler的callBack处理了消息，并没有返回true,则会回调Handler的handleMessage()方法:

```java
    public void dispatchMessage(Message msg) {
		//如果callBack不为null,则传给callBack处理。
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
			//如果Handler的callBack不为空，则传给callBack处理
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

> handleCallback()方法:

```java
    private static void handleCallback(Message message) {
		//回调Runnable的run()方法
        message.callback.run();
    }
```