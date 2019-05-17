# 基于Android的模拟点击探索

## 前言

压力测试中，一般会用到自动化测试。准备写一个APP，可以记录屏幕上的点击事件，然后通过shell命令来模拟自动执行。shell指令，比较容易实现。那么，关键的一步是获取点击的坐标。对于Android来说，为便于开发者调试，Android系统中的"开发者选项"中，有一个"指针位置"的选项。打开这个选项，点击屏幕，就会显示当前点击的位置坐标。接下来，来看一下打开选项的过程。

## 开发者选项页面

**"开发者选项"的源码位于packages/apps/settings/src/com/android/settings/DevelopmentSettings.java文件中。**

    private SwitchPreference mPointerLocation;
	
**在onCreate()方法中初始化:**

    mPointerLocation = findAndInitSwitchPref(POINTER_LOCATION_KEY);

**findAndInitSwitchPref()方法:**

```java
    private SwitchPreference findAndInitSwitchPref(String key) {
        SwitchPreference pref = (SwitchPreference) findPreference(key);
        if (pref == null) {
            throw new IllegalArgumentException("Cannot find preference with key = " + key);
        }
        mAllPrefs.add(pref);
        mResetSwitchPrefs.add(pref);
        return pref;
    }
```

**当点击选项开关切换后，会把当前的开关状态存入Settings数据库。**

```java
    private void writePointerLocationOptions() {
        Settings.System.putInt(getActivity().getContentResolver(),
                Settings.System.POINTER_LOCATION, mPointerLocation.isChecked() ? 1 : 0);
    }
```

## PhoneWindowManager

**PhoneWindowManager的源码位于framework/base/services/core/java/com/android/server/policy/PhoneWindowManager.java文件中。**

**PhoneWindowManager会监听Settings.System.POINTER_LOCATION字段的变化。**

```java
    class SettingsObserver extends ContentObserver {
        SettingsObserver(Handler handler) {
            super(handler);
        }

        void observe() {
            // Observe all users' changes
            ContentResolver resolver = mContext.getContentResolver();
            ...
            resolver.registerContentObserver(Settings.System.getUriFor(
                    Settings.System.POINTER_LOCATION), false, this,
                    UserHandle.USER_ALL);
            ...
            updateSettings();
        }

        @Override public void onChange(boolean selfChange) {
            updateSettings();
            updateRotation(false);
        }
    }
```

**当这个值发生变化时，在updateSettings()方法中调用：**

```java
    public void updateSettings() {
        ContentResolver resolver = mContext.getContentResolver();
        boolean updateRotation = false;
        synchronized (mLock) {
            ...

            if (mSystemReady) {
                int pointerLocation = Settings.System.getIntForUser(resolver,
                        Settings.System.POINTER_LOCATION, 0, UserHandle.USER_CURRENT);
                if (mPointerLocationMode != pointerLocation) {
                    mPointerLocationMode = pointerLocation;
                    mHandler.sendEmptyMessage(pointerLocation != 0 ?
                            MSG_ENABLE_POINTER_LOCATION : MSG_DISABLE_POINTER_LOCATION);
                }
            }
            ...
        }
        synchronized (mWindowManagerFuncs.getWindowManagerLock()) {
            PolicyControl.reloadFromSetting(mContext);
        }
        if (updateRotation) {
            updateRotation(true);
        }
    }
```

**在这个方法中，会通过Handler能送一个Message去处理。**

```java
    private class PolicyHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_ENABLE_POINTER_LOCATION:
                    enablePointerLocation();
                    break;
                case MSG_DISABLE_POINTER_LOCATION:
                    disablePointerLocation();
                    break;
				...
            }
        }
    }
```

**如果打开了"指针位置"的选项开关，那么会调用enablePointerLocation()方法**

```java
    private void enablePointerLocation() {
        if (mPointerLocationView == null) {
            mPointerLocationView = new PointerLocationView(mContext);
            mPointerLocationView.setPrintCoords(false);
            WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
                    WindowManager.LayoutParams.MATCH_PARENT,
                    WindowManager.LayoutParams.MATCH_PARENT);
            lp.type = WindowManager.LayoutParams.TYPE_SECURE_SYSTEM_OVERLAY;
            lp.flags = WindowManager.LayoutParams.FLAG_FULLSCREEN
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN;
            if (ActivityManager.isHighEndGfx()) {
                lp.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
                lp.privateFlags |=
                        WindowManager.LayoutParams.PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED;
            }
            lp.format = PixelFormat.TRANSLUCENT;
            lp.setTitle("PointerLocation");
            WindowManager wm = (WindowManager) mContext.getSystemService(WINDOW_SERVICE);
            lp.inputFeatures |= WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL;
            wm.addView(mPointerLocationView, lp);
            mWindowManagerFuncs.registerPointerEventListener(mPointerLocationView);
        }
    }
```

**在这个方法中，首先初始化一个PointerLocationView对象，然后设置WindowManager.LayoutParams，然后将PointerLocationView实例添加到window中。再通过WindowManagerFuncs注册监听。**

**当屏幕上有点击时，会回调PointerLocationView的onPointerEvent()方法：**

```java
    @Override
    public void onPointerEvent(MotionEvent event) {
        ...
    }
```

通过反射可以获取到PointerLocationView的实例，但是无法获取到WindowManagerFuncs实例。WindowManagerFuncs是在PhoneWindowManager的init()方法中初始化的。

```java
    @Override
    public void init(Context context, IWindowManager windowManager,
            WindowManagerFuncs windowManagerFuncs) {
        mContext = context;
        mWindowManager = windowManager;
        mWindowManagerFuncs = windowManagerFuncs;
        ...
        }
```
对于WindowManager的流程不了解。这种方法看来是行不通了。。。

在网上查了相关的资料，还有种方法是通过adb的getevent命令来获取/dev/input/路径下的event事件数据，然后解析相关数据。不过对于这块也不熟悉，就没有再深入研究。

总的来说，开发基于Android的模拟点击的应用是以失败告终。后面有时间再研究下是否有其它方法可以实现。