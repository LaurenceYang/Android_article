# OOM优化

项目前期，后台的崩溃错误列表中很大一部分都是OOM的报错，所以我们花了很大一部分时间对项目的OOM进行了优化。下面对在OOM优化中使用的方法做一个总结。



## 问题来源

经过调查，OOM的来源主要来自下面两个方面

> * 应用中有大量的图片使用
> * 存在一定程度的内存泄漏



## 大量图片引起的OOM的解决方案

> * 首先还是对使用到的图片进行优化，考虑在不影响产品体验的情况下减小图片的大小及尺寸
> * 图片加载框架的改变。刚好15年上半年facebook推出了fresco，其核心就是通过“偷”内存解决OOM的问题。我们在调研了一段时间之后在fresco开源三个月后集成到了我们的版本中，最后通过数据分析效果明显，关于fresco的原理及我们使用的效果可参考[[原]Fresco是如何"偷"内存减少OOM的]()
> * 小tips，使用fresco的隐藏接口PlatformBitmapFactory.createBitmap。fresco提供的是基于imageview加载图片时防止OOM的解决方案；但对于其它控件中使用的图片却不适用，如textview背景是一张很大的图片的情况是不能直接使用fresco的，这种情况可用PlatformBitmapFactory.createBitmap的方式生成一张图片然后设置为textview的背景。
> * 当然fresco并不是万能，其最好的效果是在4.4及其以下版本中，并且因其使用了native code导致其容量比一般的图片加载库要大，这和apk大小的优化有一定冲突，需要衡量。
> * android:largeHeap="true"，虽然google不建议这样做，但反编译看了BAT的软件，都在用，所以还是紧跟潮流吧，这样可以一定程度让我们的app多分到内存。



## 内存泄漏的解决方案

关于内存泄漏，是一个老生常谈的问题，网上也有很多内存泄漏的处理方案。这里谈下我们项目中常遇到的内存泄漏的种类。

> * Activity的内存泄漏。一个我们遇到的案例是在使用第三方的页面推出动画类时，存在了对Activity的饮用，该饮用为static类型，导致了Activity不能被GC。解决方法就是自改第三方库，去掉静态引用。
> * Activity的内存泄漏2。我们有些管理类为单例模式，在使用该管理类的时候通过getInstance(Context context)获取实例，但是传入的context为当前activity，导致该activity不能被GC。解决方法就是传入的context使用application的context。
> * Activity的内存泄漏3。使用sharepreference的地方比较混乱，有些地方使用activity的context，有些地方使用application的context。解决方法时统一使用application的context。
> * 匿名内部类，如handler。解决方法是handler使用静态内部类，handler中需要使用到如activity的地方使用弱引用。
> * toast创建时的context类型。如页面结束时toast会显示几秒，在这几秒类如果context是activity的话，activity就处于内存泄漏的状态。toast创建通常使用application的context即可。
> * 关于static类型，不是不能用，但用的时候要慎用，需要分析其是否可能会引起内存泄漏。
> * WebView的使用，其是吃内存的大户，App中需要控制WebView的使用个数，不能过多，不建议超过2个，能复用时尽量复用，不使用时关闭其使用的资源。

## 其它

> * 引入LeakCanary进行检测