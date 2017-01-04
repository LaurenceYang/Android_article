# Android热更新/资源混淆/编译/多渠道打包流程的整合

标签（空格分隔）： Android 热更新 混淆 打包

---

##问题描述
目前我们的项目集成了热更新、多渠道打包、资源混淆。其中热更新使用了[Tinker](https://github.com/Tencent/tinker)，多渠道打包使用了[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)，资源混淆使用了[AndResGuard](https://github.com/shwenzhang/AndResGuard)。如果单独使用其中一个功能时，使用它们提供的命令直接编译就可以了，但因为引入了资源混淆，热更新和多渠道打包都依赖资源混淆后生成的apk包，所以需要整合编译流程。

##整合
**整合前**
编译：生成的Release文件没有进行资源混淆
```groovy
./gradlew assembleRelease
```

资源混淆
```groovy
./gradlew resguardRelease
```

Tinker生成补丁：生成的Release文件没有进行资源混淆
```groovy
./gradlew tinkerPatchRelease
```

多渠道打包：生成多渠道文件没有进行资源混淆
```groovy
./gradlew apkRelease
```

**整合后**
主要解决两个问题
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

针对问题2、暂时使用命令行的方式进行生成渠道包，只要输入的原始文件是混淆后的即可。考虑到市场推广人员会打不同渠道包，后期可做一个简易工具提供给推广人员。

---
整合后操作：
**编译：**
```java
./gradlew resguardRelease
```
生成的apk文件放在${app}\build\bakApk\resguard\目录下
**打补丁包：**
```java
./gradlew tinkerPatchRelease
./gradlew generateManifestForReleaseTinkerPatch
```
最终生成的补丁放在${app}\build\outputs\patch\目录下

**多渠道打包：**
针对编译后生成的包，使用packer-ng提供的命令行操作即可
```groovy
java -jar PackerNg-x.x.x.jar apkFile marketFile outputDir
```
