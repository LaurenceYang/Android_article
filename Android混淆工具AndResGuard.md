# Android混淆工具AndResGuard

标签（空格分隔）： Android 开源项目 混淆

---

Github地址：https://github.com/shwenzhang/AndResGuard

AndResGuards是微信张绍文大神开源的项目，该项目可以帮助你减小APK大小，原理类似于JAVA Proguard，但是只针对于资源，它会将冗长的资源文件变短，并且不会干涉编译过程，可以直接对生成的apk文件进行混淆操作，得到一个资源混淆后的apk（若在配置文件中输入签名信息，可自动重签名并对齐，得到可直接发布的apk）以及对应资源ID的mapping文件。

##原理
我们在使用资源文件的时候通常使用的都是一个int类型的id值，而不是直接使用资源名字，所以这其中存在一张类似于Map的表，来实现这意义对应的关系。

如果我们修改掉资源名字，使其变短，变成a、b、c、d这样的名字，而对应的id不变化，是不会影响apk中使用资源的。所以AndResGuard就是通过减小文件名的长度来减小资源文件的大小。这边是AndResGuard实现的基础。

同时，在修改资源文件名的同时通过7z进行极限打包，最大程度的减少了apk文件的大小。

这种实现方式不会出问题的前提是程序中的资源都是通过id来查找的，那如果这时使用getResource.getIdentifier去查找的话，因为名字变了，肯定就找不到了，这个时候就会抛出crash，AndResGuard针对这种问题添加了一个whitelist，将不能混淆的资源文件放在whitelist就不会对其进行混淆。


##apk减小原因
> * resources.arsc变小，而且这个文件一般是不会压缩的
> * 文件信息变小；由于采用了超短路径，例如res/drawable/emoji.png被改成r/d/e
> * 签名信息会变小；由于采用了超短路径，签名过程需要对每个文件用sha1算法生成摘要信息
> * 压缩率更高：由于采用了7z极限压缩，128k的字典。要注意哪些文件不需要压缩。    

##一些需要注意的问题
> * compress参数对混淆效果的影响
若指定compess 参数.png、.gif以及*.jpg，resources.arsc会大大减少安装包体积。若要支持2.2，resources.arsc需保证压缩前小于1M。
> * 操作系统对7z的影响
实验证明，linux与mac的7z效果更好
> * keepmapping方式对增量包大小的影响
影响并不大，但使用keepmapping方式有利于保持所有版本混淆的一致性
> * 渠道包的问题(**建议通过修改zip摘要的方式生产渠道包**)
在出渠道包的时候，解压重压缩会破坏7zip的效果，通过repackage命令可用7zip重压缩。
> * 若想通过getIdentifier方式获得资源，需要放置白名单中。
部分手机桌面快捷图标的实现有问题，务必将程序桌面icon加入白名单



参考：
https://github.com/shwenzhang/AndResGuard/blob/master/doc/how_to_work.zh-cn.md  

[安装包立减1M--微信Android资源混淆打包工具](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)  

[Android逆向之旅---解析编译之后的Resource.arsc文件格式](http://blog.csdn.net/jiangwei0910410003/article/details/50628894)  

[Apktool](https://github.com/iBotPeaches/Apktool)  





