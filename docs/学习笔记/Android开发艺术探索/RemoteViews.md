# RemoteViews

>RemoteViews表示的是一个View结构，它可以在其他进程中显示。RemoteViews在Android中的使用场景有两种：通知栏和桌面小部件。

## 1、RemoteViews的应用

RemoteViews在实际开发中，主要用在通知栏的桌面小部件的开发过程中。通知栏主要是通过NotificationManager的notify方法来实现，除了默认效果，还可以另外定义布局。桌面小部件则是通过AppWidgetProvider来实现，AppWidgetProvider本质上是一个广播。RemoteViews运行在系统的SystemServer进程。

AppWidgetProvider除了最常用的onUpdate方法，还有以下几个方法：

- **onEnable:**

当该窗口小部件第一次添加到桌面时调用该方法，可添加多次但只在第一次调用。

- **onUpdate:**

小部件被添加时或者每次小部件更新时都会调用一次该方法，小部件的更新时机由updatePeriodMillis来指定，每个周期小部件都会自动更新一次。

- **onDeleted:**

每删除一次桌面小部件就调用一次

- **onDisabled:**

当最后一个该类型的桌面小部件被删除时调用该方法

- **onReceive:**

这是广播的内置方法，用于分发具体的事件给其它方法。

**PendingIntent**

PendingIntent表示一种处于pending状态的意图，而pending状态表示的是一种待定、等待、即将发生的意思，就是说接下来有一个Intent将在某个特定的时刻发生。PendingIntent和Intent的区别在于，PendingIntent是在将来的某个不确定的时刻发生，而Intent是立刻发生。PendingIntent典型使用场景是给RemoveViews添加单击事件，通过send和cancel方法来发送和取消特定的待定的Intent。

PendingIntent主要方法：

**getActivity(Context context,int requestCode,Intent intent,int flags)**

>获得一个PendingIntent，该待定意图发生时，效果相当于Context.startActivity(Intent)

**getService(Context context,int requestCode,Intent intent,int flags)**

>获得一个PendingIntent，该待定意图发生时，效果相当于Context.startService(Intent)

**getBroadcast(Context context,int requestCode,Intent intent,int flags)**

>获得一个PendingIntent，该待定意图发生时，效果相当于Context.sendBroadcast(Intent)

PendingIntent匹配规则：如果两个PendingIntent它们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就是相同的。

**flags:**

- FLAG_ONE_SHOT:

当前描述的PendingIntent只能被使用一次，然后它就会被自动cancel。

- FLAG_NO_CREATE:

当前描述的PendingIntent不会主动创建。日常开发中没有太多的使用意义。

- FLAG_CANCEL_CURRENT:

当前描述的PendingIntent如果已经存在，那么它们都会被cancel，然后系统创建一个新的PendingIntent。对于通知栏消息，那些被cancel的消息单击后将无法打开。

- FLAG_UPDATE_CURRENT:

当前描述的PendingIntent如果已经存在，那么它们都会被更新，即Intent中的Extras会被替换成最新的。

## 2、RemoteViews的内部机制

RemoteView并不能支持所有的View类型，它所支持的类型如下：

**Layout**

FrameLayout、LinearLayout、RelativeLayout、GridLayout。

**View**

AnalogClock、Buttom、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper、ListView、GridView、StackView、AdapterViewFlipper、ViewStub。

上面所描述的是RemoteViews所支持的所有View类型，RemoteViews不支持它们的子类以及其他View类型。

通知栏和桌面小部件分别由NotificationManager和AppWidgetManager管理，而NotificationManager和AppWidgetManager通过Binder分别和SystemServer进程中的NotificationManagerService以及AppWidgetService进行通信。由此可见，通知栏和桌面小部件中的布局实际是在NotificationManagerService以及AppWidgetService中被加载的,而它们运行在系统的SystemServer中。

setOnClickPendingIntent、setPendingIntentTemplate以及setOnClickFillInIntent它们之间的区别和联系：

首先setOnClickPendingIntent用于给普通View设置单击事件，但不能给集合(ListView和StackView)中的View设置单击事件，比如我们不能给ListView中的item通过setOnClickPendingIntent这种方式添加单击事件，因为开销比较大，所以系统禁止了这种方式；其次，如果要给ListView和StackView中的item添加单击事件，则必须将setPendingIntentTemplate和setOnClickFillInIntent组合使用才可以。