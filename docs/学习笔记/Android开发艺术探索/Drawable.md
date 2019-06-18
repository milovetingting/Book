# Drawable

>Drawable表示的是一种可以在Canvas上进行绘制的抽象的概念。

## 1、Drawable简介

Drawable是一个抽象类，是所有Drawable对象的基类，每个具体的Drawable都是它的子类，比如ShapeDrawable、BitmapDrawable等。

通过getIntrinsicWidth和getIntrinsicHeight这两个方法可以获取到Drawable的内部宽/高。但并不是所有的Drawable都有内部宽/高，比如一张图片所形成的Drawable，它的内部宽/高就是图片的宽/高，但是一个颜色形成的Drawable，它没有内部宽/高。Drawable内部宽/高不等同于它的大小。Drawable是没有大小概念的。

## 2、Drawable分类

- BitmapDrawable

- ShapeDrawable

- LayerDrawable

对应的XML标签是<layer-list>,表示一种层次化的Drawable集合。

- StateListDrawable

对应于<selector>标签，也是表示Drawable集合，每个Drawable都对应View的一种状态。主要用于设置可单击的View的背景。

系统会根据View当前的状态从selector中选择对应的item,每个item对应着一个具体的Drawable，系统按照从上到下的顺序查找，直至查找到第一条匹配的item。一般来说，默认的item都应该放在selector的最后一条并且不带任何状态，这样当上面的item无法匹配View的当前状态，系统会选择默认的item。

- LevelListDrawable

对应于<level-list>标签，表示一个Drawable集合，集合中的每个Drawable都有一个等级level的概念。根据不同的等级，LevelListDrawable会切换为对应的Drawable。Drawable等级是有范围的，即0-10000，最小等级为0，这也是默认值，最大等级为10000。

- TransitionDrawable

对应于<transition>标签，用于实现两个Drawable之间的淡入淡出效果。

- InsetDrawable

对应于<inset>标签，它可以将其它Drawable内嵌到自己当中，并可以在四周留出一定的间距。当一个View希望自己的背景比自己的实际区域小的时候，可以采用InsetDrawable来实现。通过LayerDrawable也可以实现这种效果。

- ScaleDrawable

ScaleDrawable对应于<scale>标签，可以根据自己的等级将指定的Drawable缩放到一定比例。等级为0表示ScaleDrawable不可见，这是默认值，要想ScaleDrawable可见，需要等级不为0。

- ClipDrawable

对应于<clip>标签,可以根据自己当前的等级来裁剪另一个Drawable，裁剪方向可以通过android:clipOrientation和android:gravity这两个属性来共同控制。

## 3、自定义Drawable