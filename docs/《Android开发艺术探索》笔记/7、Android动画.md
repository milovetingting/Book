# Android动画

Android的动画可以分为三种:View动画、帧动画和属性动画，帧动画也属于View动画的一种，只不过它和平移、旋转等常见的View动画在表现形式上略有不同而已。

## 1、View动画

- 平移动画：TranslateAnimation

- 缩放动画：ScaleAnimation

- 旋转动画：RotateAnimation

- 透明度动画：AlphaAnimation

用XML来定义属性动画需要定义在res/anim目录下。

## 2、View动画的特殊使用场景

### 2.1、LayoutAnimation

LayoutAnimation使用于ViewGroup，为ViewGroup指定一个动画，它的子元素出场时都会具有这种动画效果。这种效果常用在ListView上。

### 2.2、Activity的切换效果

主要用到overridePendingTransition(int enterAnim,int exitAnim)这个方法，必须在startActivity(intent)或者finish()之后被调用才能生效。

Fragment中添加切换动画，可以通过FragmentTransaction中的setCustomAnimations()方法来添加切换动画，这个切换动画需要的是View动画。

## 3、属性动画

属性动画可以对任意对象的属性进行动画而不仅仅是View，动画默认时间间隔是300ms，默认帧是10ms/帧。

用XML来定义属性动画需要定义在res/animator目录下。

对object的属性abc属性做动画，如果要让动画生效，要同时满足两个条件:

1、object必须要提供setAbc方法，如果动画的时候没有传递初始值，还要提供getAbc方法，因为系统要去取abc属性的初始值。如果不满足这条，程序直接Crash。

2、object的setAbc对属性abc所做的改变必须能够通过某种方法反映出来，比如会带来UI的改变等。如果不满足这条，动画无效果但不会Crash。



如果只满足条件1，不满足条件2，可以有3种解决方法：

- 给对象加上get和set，如果有权限的话

- 用一个类来包装原始对象，间接为其提供get和set方法

```java
	private void performAnimate(){
		ViewWrapper wrapper = new ViewWrapper(mButton);
		ObjectAnimator.ofInt(wrapper,"width",500).setDuration(5000).start();
	}
	
	private static class ViewWrapper{
		private View mTarget;
		
		public ViewWrapper(View target){
			mTarget = target;
		}

		public int getWidth(){
			return mTarget.getLayoutParams().width;
		}

		public void setWidth(int width){
			mTarget.getLayoutParams().width = width;
			mTarget.requestLayout();
		}
	}
```

- 采用ValueAnimator，监听动画过程，自己实现属性的改变

## 4、注意事项

1、OOM问题

这个问题主要出现在帧动画中，当图片数量较多肯于图片较大时就极易出现OOM。

2、内存泄露

在属性动画中有一类无限循环的动画，这类动画需要在Activity退出时及时停止，否则将导致Activity无法释放从而造成内存泄露，View动画不存在此问题。

3、兼容性问题

动画在3.0以下系统上有兼容性问题。

4、View动画的问题

View动画是对View的影像做动画，并不是真正地改变View的状态，因此有时候会出现动画完成后View无法隐藏的现象，即setVisibility(View.GONE)失效，这时只要调用view.clearAnimation()清除View动画即可解决此问题。

5、不要用px

6、动画元素的交互

7硬件加速

使用动画的过程中，建议开启硬件加速，这样会提高动画的流畅性。