# Android热更新开源项目Tinker集成实践总结

标签（空格分隔）： Android 开源项目 热更新

---

##前言
最近项目集成了Tinker，开始认为集成会比较简单，但是在实际操作的过程中还是遇到了一些问题，本文就会介绍在集成过程大家基本会遇到的主要问题。

##考虑一：后台的选取
目前后台功能可以通过三种方式实现：
1、自己搭建后台布丁下发系统
2、第三方提供的服务，目前如原微信simsun大神的个人tinkerpatch平台，目前出于内测阶段，暂时免费。后期应该会按下发量对app进行收费。
3、腾讯Bugly提供的服务，提供了热更新的下发后台，集成到了bugly的升级sdk中。免费。
根据公司的精神，我们选择了Bugly作为我们的方案，这个大家都懂得。

##考虑二：多渠道打包的问题
我们有将近100个渠道，每个渠道需要一个不同的渠道号，按product flavor的方式打出来的包的dex都有差异。这样就造成100个渠道包的热更新就需要100个补丁，这对管理简直是一个灾难。
Tinker也对这种问题给出了推荐的方案，那就是使用开源项目packer-ng-plugin，它的原理是将渠道信息写在apk文件的zip comment中，这样在多渠道打包时就不会影响dex的内容。具体关于packer-ng-plugin的介绍，可以参考文档[Android打包工具packer-ng-plugin](https://github.com/LaurenceYang/Android_article/blob/master/%E7%BC%96%E8%AF%91%E6%89%93%E5%8C%85/Android%E6%89%93%E5%8C%85%E5%B7%A5%E5%85%B7packer-ng-plugin.md)。

##考虑三：资源混淆所造成的问题
目前项目使用了资源混淆项目AndResGuard，关于AndResGuard的介绍，可以参考文档AndResGuard[Android混淆工具AndResGuard](https://github.com/LaurenceYang/article/blob/master/Android%E6%B7%B7%E6%B7%86%E5%B7%A5%E5%85%B7AndResGuard.md)
也正是引入了资源混淆，热更新和多渠道打包都必须依赖资源混淆后生成的apk包才行。所以我们对编译流程进行了整合。

**整合前**  

编译：编译直接使用AndResGuard提供的命令resguardRelease生成即可。注意：packer-ng提供的apkRelease命令生成的apk文件是没有资源混淆的。
```groovy
./gradlew resguardRelease
```

Tinker生成补丁：直接调用tinkerPatchRelease任务生成的Release文件没有进行资源混淆
```groovy
./gradlew tinkerPatchRelease
```

多渠道打包：使用packer-ng的命令apkRelease生成多渠道文件没有进行资源混淆
```groovy
./gradlew apkRelease
```

**整合后**  

主要解决两个问题：
1、Tinker生成补丁的原始和新的apk，需要使用资源混淆后的apk
2、多渠道打包所使用的原始apk，需要使用资源混淆后的apk

针对问题1，当使用resguardRelease进行编译，在编译完成后，将生成的apk文件、R文件、map文件和resouce map文件拷贝到${buildDir}/bakApk/resguard目录下；当使用tinkerPatchRelease生成补丁时，在tinkerPatchRelease任务前加入resguardTask
任务，这样生成补丁时使用的新旧apk都是资源混淆过的。核心的gradle代码如下：
```groovy
android.applicationVariants.all { variant ->
/**
 * task type, you want to bak
 */
def taskName = variant.name

tasks.all {
    if (variant.buildType.name == 'release') {

        if ("tinkerPatch${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

            // find resguard task
            def resguardTask
            tasks.all {
                if (it.name.startsWith("resguard")) {
                    resguardTask = it
                }
            }
            it.doFirst({
                // change build apk path
                it.buildApkPath = "${buildDir}/outputs/apk/AndResGuard_${project.getName()}-${taskName}/${project.getName()}-${taskName}_signed.apk"
            })

            // change task dependence to resguard task
            it.dependsOn resguardTask
        }

        if ("resguard${taskName.capitalize()}".equalsIgnoreCase(it.name)) {
            it.doLast {
                copy {
                    def date = new Date().format("MMdd-HH-mm-ss")
                    from "${buildDir}/outputs/apk/AndResGuard_${project.getName()}-${taskName}/${project.getName()}-${taskName}_signed_7zip_aligned.apk"
                    into file(bakPath.absolutePath + "/resguard")
                    rename { String fileName ->
                        fileName.replace("${project.getName()}-${taskName}_signed_7zip_aligned.apk", "${project.getName()}-${taskName}-${date}.apk")
                    }

                    from "${buildDir}/outputs/mapping/${taskName}/mapping.txt"
                    into file(bakPath.absolutePath + "/resguard")
                    rename { String fileName ->
                        fileName.replace("mapping.txt", "${project.getName()}-${taskName}-${date}-mapping.txt")
                    }

                    from "${buildDir}/intermediates/symbols/${taskName}/R.txt"
                    into file(bakPath.absolutePath + "/resguard")
                    rename { String fileName ->
                        fileName.replace("R.txt", "${project.getName()}-${taskName}-${date}-R.txt")
                    }
                    from "${buildDir}/outputs/apk/AndResGuard_${project.getName()}-${taskName}/resource_mapping_${project.getName()}-release.txt"
                    into file(bakPath.absolutePath + "/resguard")
                    rename { String fileName ->
                        fileName.replace("resource_mapping_${project.getName()}-release.txt", "${project.getName()}-${taskName}-${date}-resource_mapping.txt")
                    }
                }
            }
        }
    }
}
```
可根据业务进行修改  

针对问题2、在AS中使用apkRelease任务打包的方式不再适用，可直接使用packer-ng所提供的命令行方式进行生成渠道包，经过测试，100个渠道包的确在10s左右就能打完，速度相当之快。考虑到市场推广人员会打不同渠道包，后期可做一个简易工具提供给市场推广人员。

整合后操作：  

**编译：**  

```java
./gradlew resguardRelease
```
生成的apk文件放在${app}\build\bakApk\resguard\目录下  

**打补丁包：**  
```java
./gradlew tinkerPatchRelease  //mac下不需要执行下一个命令
./gradlew generateManifestForReleaseTinkerPatch
```
最终生成的补丁放在${app}\build\outputs\patch\目录下

**多渠道打包：**  

针对编译后生成的包，使用packer-ng提供的命令行操作即可
```groovy
java -jar PackerNg-x.x.x.jar apkFile marketFile outputDir
```











