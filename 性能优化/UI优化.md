# UI优化



关于UI的优化主要分为两部分：

> * 减少UI层级
> * 避免OverDraw



## 减少UI层级

> * 可使用Hierarchy Viewer查看View嵌套层级，合理较少嵌套数量



## OverDraw

OverDraw是在UI优化中效果最显著的部分。过度绘制可通过开发者选项中显示过度绘制区域进行确认。</br>

该工具会使用三种不同的颜色绘制屏幕，来指示overdraw发生在哪里以及程度如何，其中：</br>
没有颜色： 意味着没有overdraw。像素只画了一次。</br>
蓝色： 意味着overdraw 1倍。像素绘制了两次。大片的蓝色还是可以接受的（若整个窗口是蓝色的，可以摆脱一层）。</br>
绿色： 意味着overdraw 2倍。像素绘制了三次。中等大小的绿色区域是可以接受的但你应该尝试优化、减少它们。</br>
浅红： 意味着overdraw 3倍。像素绘制了四次，小范围可以接受。</br>
暗红： 意味着overdraw 4倍。像素绘制了五次或者更多。这是错误的，要修复它们。</br>



避免OverDraw的方法：</br>

> * 去掉window的默认背景。在activity的onCreate中调用getWindow().setBackgroundDrawable(null);
> * 去掉不必要的背景。如selector的背景，将normal状态的color设置为“@android:color/transparent"等
> * 合理使用控件。如LinearLayout易用，效率高，表达能力有限。RelativeLayout复杂，表达能力强，效率稍逊。
> * ViewStub。
> * Merge标签。可干掉一个view层级
> * 慎用Alapha。做Alpha转化就需要对当前View绘制两遍，可想而知，绘制效率会大打折扣，耗时会翻倍，所以Alpha还是慎用。
> * ClipRect & QuickReject。
> * 避免“OverDesign”





参考：http://www.jianshu.com/p/145fc61011cd