# APK大小优化



APK的大小会对市场人员的推广效果产生影响，所以APK的大小是越小越好。

下面对我们在项目中对apk大小优化做一个总结：



## 代码优化

> * 去掉不使用的代码
> * 重复代码需要抽出来
> * 使用第三方库时要谨慎，考虑其对apk大小的影响
> * 使用代码混淆Proguard
> * so文件考虑其支持的平台，一般情况下支持arm即可。x86，mips等不常用架构可考虑拿掉
> * 代码规范，如成员变量在定义时不需要赋null值，如A a = null可修改为A a，可一定程度上减小大小；



## 资源优化

> * 每次发版本前使用Lint工具查找没有使用的资源，去掉它们
> * UI提供的图片要求他们最大程度的压缩
> * 图片格式上面，有些大图片考虑实际情况可以使用jpg替换png
> * 纯颜色的图片，尽量使用xml文件实现
> * 9.png图片，可拉伸区域尽量做到最小，能使用9.png图片进行拉伸的地方不要使用全图片
> * 进行资源混淆，如AndResGuard，可参考文档[[原]Android混淆工具AndResGuard解析](https://github.com/LaurenceYang/article/blob/master/%E7%BC%96%E8%AF%91%E6%89%93%E5%8C%85/Android%E6%B7%B7%E6%B7%86%E5%B7%A5%E5%85%B7AndResGuard.md)。我们的apk经过混淆后，从原来的7M减小到6.4M
> * 可在编译时将初R$styleable.class以外的所有R.class删除掉，并且在应用的地方替换成相应的常量，从而达到缩减包大小和减少dex个数的效果，可参考[ThinRPlugin](https://github.com/meili/ThinRPlugin)
> * 有选择性的提供hdpi，xhdpi，xxhdpi的图片资源。建议优先提供xhdpi的图片，对于mdpi，ldpi与xxxhdpi根据需要提供有差异的部分即可
> * 考虑使用webp或者svg格式



## 其它

> * 删除不要的功能，迷之微笑，但是终极奥秘
> * 某些引入的第三方库，如果只是使用其一小部分功能，可考虑删减库大小
> * 动态加载，某些功能可以通过使用时下载补丁，动态加载的方式，从而减小app的大小
> * 插件技术