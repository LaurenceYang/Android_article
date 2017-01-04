# 热更新开源项目Tinker源码解析之Library热更新

标签（空格分隔）： Android 开源项目 热更新

---

Tinker的Library热更新原理相对于Dex和资源的更新，简单很多。生成补丁时根据BSdiff算法生成补丁包，然后在下发补丁成功后根据BSpatch算法将补丁包和旧的library合成新的library，并将更新后的Library库文件保存在tinker下面的目录下。具体的源码不再做阐述。

但是Tinker并没有直接将补丁的lib路径添加到`DexPathList`中，理论上这样可以做到程序完全没有感知的对Library文件作补丁。这里主要是因为在多abi的情况下，某些机器获取的并不准确。所以想要加载最新的库，需要自己使用`TinkerInstaller.load*Library`去加载库文件，它会自动尝试先去Tinker中的库文件加载，加载不成功会调用`System.loadLibrary`调用系统的库文件。
```java
//load lib/armeabi library
TinkerInstaller.loadArmLibrary(getApplicationContext(), "stlport_shared");
//load lib/armeabi-v7a library
TinkerInstaller.loadArmV7Library(getApplicationContext(), "stlport_shared");
```

对于第三方库文件的加载，我们无法干预其加载时机，但是只要在我们的代码提前加载第三方的库文件即可。若想对第三方代码的库文件更新，可先使用`TinkerInstaller.load*Library`对第三方库做提前的加载！

当前使用方式似乎并不能做到开发者透明，这是因为Tinker想尽量少的去hook系统框架减少兼容性的问题。