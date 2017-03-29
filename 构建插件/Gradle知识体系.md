# Gradle知识体系

---
## 基本项目
### 简单的构建文件
一个Gradle项目的构建过程定义在build.gradle文件中，位于项目的根目录下。</br>

一个最简单的Gradle纯java项目的build.gradle文件包含以下内容</br>
```
apply plugin: 'java'
```
这里引入了Gradle的Java插件。这个插件提供了所有构建和测试Java应用程序所需要的东西。</br>
</br>
最简单的Android项目的build.gradle文件包含以下内容
```
buildscript {
    repositories {
        mavenCentral()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:0.11.1'
    }
}

apply plugin: 'android'

android {
    compileSdkVersion 19
    buildToolsVersion "19.0.0"
}
```

这里包括了Android build file的3个主要部分：</br>

`buildscript{...}`这里配置了驱动构建过程的代码。</br>
在这个部分，它声明了使用Maven仓库，并且声明了一个maven文件的依赖路径。这个文件就是包含了0.11.1版本android gradle插件的库。</br>

>注意：这里的配置只影响控制构建过程的代码，不影响项目源代码。项目本身需要声明自己的仓库和依赖关系，稍后将会提到这部分。

接下来，跟前面提到的Java Plugin一样添加了android plugin。</br>

最后，`andorid{...}`配置了所有android构建过程需要的参数。**这里也是Android DSL的入口点**。</br>
默认情况下，只需要配置目标编译SDK版本和编译工具版本，即compileSdkVersion和buildToolsVersion属性。 这个complieSdkVersion属性相当于旧构建系统中project.properites文件中的target属性。这个新的属性可以跟旧的target属性一样指定一个int或者String类型的值。</br>

>重要：你只能添加android plugin。同时添加java plugin会导致构建错误。

>注意：你同样需要在相同路径下添加一个local.properties文件，并使用sdk.dir属性来设置SDK路径。 另外，你也可以通过设置ANDROID_HOME环境变量，这两种方式没有什么不同，根据你自己的喜好选择其中一种设置。


### 项目结构
上面提到的基本的构建文件需要一个默认的文件夹结构。Gradle遵循约定优先于配置的概念，在可能的情况尽可能提供合理的默认配置参数。

基本的项目开始于两个名为“source sets”的组件，即main source code和test code。它们分别位于：

> * src/main/
> * src/androidTest/

里面每一个存在的文件夹对应相应的源组件。 对于Java plugin和Android plugin来说，它们的Java源代码和资源文件路径如下：

> * java/
> * resources/

但对于Android plugin来说，它还拥有以下特有的文件和文件夹结构：

> * AndroidManifest.xml
> * res/
> * assets/
> * aidl/
> * rs/
> * jni/

### 构建变种版本
Build Type + Product Flavor = Build Variant（构建类型+定制产品=构建变种版本）

### 高级构建定制
#### Build options（构建选项）
* Java Compilation options（Java编译选项）
```
android {
    compileOptions {
        sourceCompatibility = "1.6"
        targetCompatibility = "1.6"
    }
}
```
默认值是“1.6”。这个设置将影响所有task编译Java源代码。

* aapt options（aapt选项）
```
android {
    aaptOptions {
        noCompress 'foo', 'bar'
        ignoreAssetsPattern "!.svn:!.git:!.ds_store:!*.scc:.*:<dir>_*:!CVS:!thumbs.db:!picasa.ini:!*~"
    }
}
```
这将影响所有使用aapt的task。

* dex options（dex选项）
```
android {
    dexOptions {
        incremental false

        preDexLibraries = false

        jumboMode = false

    }
}
``` 
这将应用所有使用dex的task。

#### Manipulation tasks（操作task）
基础Java项目有一组有限的task用于互相处理生成一个输出。 classes是一个编译Java源代码的task。可以在build.gradle文件中通过脚本很容易使用classes。这是project.tasks.classes的缩写。

在Android项目中，相比之下这就有点复杂。因为Android项目中会有大量相同的task，并且它们的名字基于Build Types和Product Flavor生成。
为了解决这个问题，android对象有两个属性：

* applicationVariants（只适用于app plugin）
* libraryVariants（只适用于library plugin）
* testVariants（两个plugin都适用）

这三个都会分别返回一个ApplicationVariant、LibraryVariant和TestVariant对象的DomainObjectCollection。

注意使用这三个collection中的其中一个都会触发生成所有对应的task。这意味着使用collection之后不需要更改配置。

DomainObjectCollection可以直接访问所有对象，或者通过过滤器进行筛选。
```
android.applicationVariants.each { variant ->
    ....
}
```

每个task类型的API由于Gradle的工作方式和Android plugin的配置方式而受到限制。 首先，Gradle意味着拥有的task只能配置输入输出的路径和一些可能使用的选项标识。因此，task只能定义一些输入或者输出。

其次，这里面大多数task的输入都不是单一的，一般都混合了sourceSet、Build Type和Product Flavor中的值。为了保持构建文件的简单和可读性，目标是要让开发者通过DSL语言修改这些对象来配饰构建的过程，而不是深入修改输入和task的选项。

另外需要注意，除了ZipAlign这个task类型，其它所有类型都要求设置私有数据来让它们运行。这意味着不可能自动创建这些类型的新task实例。

这些API也可能会被更改。一般来说，目前的API是围绕着给定task的输入和输出入口来添加额外的处理（如果需要的时候）。欢迎反馈意见，特别是那些没有预见过的需求。




</br>
## 参考
[Gradle Plugin UserGuide](http://tools.android.com/tech-docs/new-build-system/user-guide)
[Gradle Android plugin插件用户指南翻译](http://avatarqing.github.io/Gradle-Plugin-User-Guide-Chinese-Verision/)
[Android Plugin DSL Reference](http://google.github.io/android-gradle-dsl/current/)
[Gradle User Guide](https://docs.gradle.org/current/userguide/userguide_single.html)
[The Java Plugin](https://docs.gradle.org/current/userguide/java_plugin.html#N12394)



