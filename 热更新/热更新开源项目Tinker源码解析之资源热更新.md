# 热更新开源项目Tinker源码解析之资源热更新

标签（空格分隔）： Android 开源项目 热更新

---

上一篇文章介绍了Dex文件的热更新流程，本文将会分析inker中对资源文件的热更新流程，同Dex，资源文件同样包括三个部分：资源补丁生成，资源补丁合成及资源补丁加载。

##一、资源补丁生成
ResDiffDecoder.patch(File oldFile, File newFile)
```java
// 如果是新增的资源，直接将资源文件拷贝到目标目录.
if (oldFile == null || !oldFile.exists()) {
    if (Utils.checkFileInPattern(config.mResIgnoreChangePattern, name)) {
        Logger.e("found add resource: " + name + " ,but it match ignore change pattern, just ignore!");
        return false;
    }
    FileOperation.copyFileUsingStream(newFile, outputFile);
    addedSet.add(name);
    writeResLog(newFile, oldFile, TypedValue.ADD);
    return true;
}
...
// 新旧资源文件的md5一样，表示没有修改.
if (oldMd5 != null && oldMd5.equals(newMd5)) {
    return false;
}
...
// 修改的资源文件使用dealWithModeFile函数处理.
dealWithModeFile(name, newMd5, oldFile, newFile, outputFile);
```

dealWithModeFile会对文件大小进行判断，如果大于设定值（默认100Kb），采用bsdiff算法对新旧文件降低补丁包的大小。如果小于设定值，将该文件加入修改列表，并直接将新文件拷贝到目标目录。
```java
if (checkLargeModFile(newFile)) { //大文件采用bsdiff算法
    if (!outputFile.getParentFile().exists()) {
        outputFile.getParentFile().mkdirs();
    }
    BSDiff.bsdiff(oldFile, newFile, outputFile);
    //treat it as normal modify
    // 对生成的diff文件大小和newFile进行比较，只有在达到我们的压缩效果后才使用diff文件
    if (Utils.checkBsDiffFileSize(outputFile, newFile)) {
        LargeModeInfo largeModeInfo = new LargeModeInfo();
        largeModeInfo.path = newFile;
        largeModeInfo.crc = FileOperation.getFileCrc32(newFile);
        largeModeInfo.md5 = newMd5;
        largeModifiedSet.add(name);
        largeModifiedMap.put(name, largeModeInfo);
        writeResLog(newFile, oldFile, TypedValue.LARGE_MOD);
        return true;
    }
}
modifiedSet.add(name); // 加入修改列表
FileOperation.copyFileUsingStream(newFile, outputFile);
writeResLog(newFile, oldFile, TypedValue.MOD);
return false;
```
BSDiff的具体实现我们在后面讨论。

ResDiffDecoder.onAllPatchesEnd()中会加入一个测试用的资源文件，放在assets目录下，并且向res_meta.txt文件中写入资源更改的一些信息。
```java
//加入一个测试用的资源文件
addAssetsFileForTestResource();
...
//first, write resource meta first
//use resources.arsc's base crc to identify base.apk
String arscBaseCrc = FileOperation.getZipEntryCrc(config.mOldApkFile, TypedValue.RES_ARSC);
String arscMd5 = FileOperation.getZipEntryMd5(extractToZip, TypedValue.RES_ARSC);
if (arscBaseCrc == null || arscMd5 == null) {
    throw new TinkerPatchException("can't find resources.arsc's base crc or md5");
}

String resourceMeta = Utils.getResourceMeta(arscBaseCrc, arscMd5);
writeMetaFile(resourceMeta);

//pattern
String patternMeta = TypedValue.PATTERN_TITLE;
HashSet<String> patterns = new HashSet<>(config.mResRawPattern);
//we will process them separate
patterns.remove(TypedValue.RES_MANIFEST);

writeMetaFile(patternMeta + patterns.size());
//write pattern
for (String item : patterns) {
    writeMetaFile(item);
}
//write meta file, write large modify first
writeMetaFile(largeModifiedSet, TypedValue.LARGE_MOD);
writeMetaFile(modifiedSet, TypedValue.MOD);
writeMetaFile(addedSet, TypedValue.ADD);
writeMetaFile(deletedSet, TypedValue.DEL);
```
最后的res_meta.txt文件的格式范例如下：
>resources_out.zip,4019114434,6148149bd5ed4e0c2f5357c6e2c577d6
pattern:4
resources.arsc
r/*
res/*
assets/*
modify:1
r/g/ag.xml
add:1
assets/only_use_to_test_tinker_resource.txt

到此，资源文件的补丁打包流程结束。

##二、补丁下发成功后资源补丁的合成
ResDiffPatchInternal.tryRecoverResourceFiles会调用extractResourceDiffInternals进行补丁的合成。
```java
// 首先读取res_meta.txt的数据
ShareResPatchInfo.parseAllResPatchInfo(meta, resPatchInfo);
// 验证resPatchInfo的MD5是否合法
if (!SharePatchFileUtil.checkIfMd5Valid(resPatchInfo.resArscMd5)) {
...
// resources.apk
File resOutput = new File(directory, ShareConstants.RES_NAME);

// 该函数里面会对largeMod的文件进行合成，合成的算法也是采用bsdiff
if (!checkAndExtractResourceLargeFile(context, apkPath, directory, patchFile, resPatchInfo, type, isUpgradePatch)) {

// 基于oldapk，合并补丁后将这些资源文件写入resources.apk文件中
while (entries.hasMoreElements()) {
    TinkerZipEntry zipEntry = entries.nextElement();
    if (zipEntry == null) {
        throw new TinkerRuntimeException("zipEntry is null when get from oldApk");
    }
    String name = zipEntry.getName();
    if (ShareResPatchInfo.checkFileInPattern(resPatchInfo.patterns, name)) {
        //won't contain in add set.
        if (!resPatchInfo.deleteRes.contains(name)
            && !resPatchInfo.modRes.contains(name)
            && !resPatchInfo.largeModRes.contains(name)
            && !name.equals(ShareConstants.RES_MANIFEST)) {
            ResUtil.extractTinkerEntry(oldApk, zipEntry, out);
            totalEntryCount++;
        }
    }
}

//process manifest
TinkerZipEntry manifestZipEntry = oldApk.getEntry(ShareConstants.RES_MANIFEST);
if (manifestZipEntry == null) {
    TinkerLog.w(TAG, "manifest patch entry is null. path:" + ShareConstants.RES_MANIFEST);
    manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, ShareConstants.RES_MANIFEST, type, isUpgradePatch);
    return false;
}
ResUtil.extractTinkerEntry(oldApk, manifestZipEntry, out);
totalEntryCount++;

for (String name : resPatchInfo.largeModRes) {
    TinkerZipEntry largeZipEntry = oldApk.getEntry(name);
    if (largeZipEntry == null) {
        TinkerLog.w(TAG, "large patch entry is null. path:" + name);
        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type, isUpgradePatch);
        return false;
    }
    ShareResPatchInfo.LargeModeInfo largeModeInfo = resPatchInfo.largeModMap.get(name);
    ResUtil.extractLargeModifyFile(largeZipEntry, largeModeInfo.file, largeModeInfo.crc, out);
    totalEntryCount++;
}

for (String name : resPatchInfo.addRes) {
    TinkerZipEntry addZipEntry = newApk.getEntry(name);
    if (addZipEntry == null) {
        TinkerLog.w(TAG, "add patch entry is null. path:" + name);
        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type, isUpgradePatch);
        return false;
    }
    ResUtil.extractTinkerEntry(newApk, addZipEntry, out);
    totalEntryCount++;
}

for (String name : resPatchInfo.modRes) {
    TinkerZipEntry modZipEntry = newApk.getEntry(name);
    if (modZipEntry == null) {
        TinkerLog.w(TAG, "mod patch entry is null. path:" + name);
        manager.getPatchReporter().onPatchTypeExtractFail(patchFile, resOutput, name, type, isUpgradePatch);
        return false;
    }
    ResUtil.extractTinkerEntry(newApk, modZipEntry, out);
    totalEntryCount++;
}

//最后对resouces.apk文件进行MD5检查，判断是否与resPatchInfo中的MD5一致
boolean result = SharePatchFileUtil.checkResourceArscMd5(resOutput, resPatchInfo.resArscMd5);
```

到此，resources.apk文件生成完毕。

##三、资源补丁加载
合成好的资源补丁存放在/data/data/${PackageName}/tinker/res/中，名为reosuces.apk。资源补丁的加载的操作主要放在TinkerResourceLoader.loadTinkerResources函数中，同dex的加载时机一样，在app启动时会被调用。直接上源码，loadTinkerResources会调用monkeyPatchExistingResources执行实际的补丁加载。
```java
public static boolean loadTinkerResources(Context context, boolean tinkerLoadVerifyFlag, String directory, Intent intentResult) {
    if (resPatchInfo == null || resPatchInfo.resArscMd5 == null) {
        return true;
    }
    String resourceString = directory + "/" + RESOURCE_PATH +  "/" + RESOURCE_FILE;
    File resourceFile = new File(resourceString);
    long start = System.currentTimeMillis();

    if (tinkerLoadVerifyFlag) {
        if (!SharePatchFileUtil.checkResourceArscMd5(resourceFile, resPatchInfo.resArscMd5)) {
            Log.e(TAG, "Failed to load resource file, path: " + resourceFile.getPath() + ", expect md5: " + resPatchInfo.resArscMd5);
            ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_RESOURCE_MD5_MISMATCH);
            return false;
        }
        Log.i(TAG, "verify resource file:" + resourceFile.getPath() + " md5, use time: " + (System.currentTimeMillis() - start));
    }
    try {
        TinkerResourcePatcher.monkeyPatchExistingResources(context, resourceString);
        Log.i(TAG, "monkeyPatchExistingResources resource file:" + resourceString + ", use time: " + (System.currentTimeMillis() - start));
    } catch (Throwable e) {
        Log.e(TAG, "install resources failed");
        //remove patch dex if resource is installed failed
        try {
            SystemClassLoaderAdder.uninstallPatchDex(context.getClassLoader());
        } catch (Throwable throwable) {
            Log.e(TAG, "uninstallPatchDex failed", e);
        }
        intentResult.putExtra(ShareIntentUtil.INTENT_PATCH_EXCEPTION, e);
        ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_RESOURCE_LOAD_EXCEPTION);
        return false;
    }

    return true;
}
```

monkeyPatchExistingResources中实现了对外部资源的加载。
```java
public static void monkeyPatchExistingResources(Context context, String externalResourceFile) throws Throwable {
    if (externalResourceFile == null) {
        return;
    }
    // Find the ActivityThread instance for the current thread
    Class<?> activityThread = Class.forName("android.app.ActivityThread");
    Object currentActivityThread = getActivityThread(context, activityThread);

    for (Field field : new Field[]{packagesFiled, resourcePackagesFiled}) {
        Object value = field.get(currentActivityThread);

        for (Map.Entry<String, WeakReference<?>> entry
            : ((Map<String, WeakReference<?>>) value).entrySet()) {
            Object loadedApk = entry.getValue().get();
            if (loadedApk == null) {
                continue;
            }
            if (externalResourceFile != null) {
                resDir.set(loadedApk, externalResourceFile);
            }
        }
    }
    // Create a new AssetManager instance and point it to the resources installed under
    // /sdcard
    // 通过反射调用AssetManager的addAssetPath添加资源路径
    if (((Integer) addAssetPathMethod.invoke(newAssetManager, externalResourceFile)) == 0) {
        throw new IllegalStateException("Could not create new AssetManager");
    }

    // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
    // in L, so we do it unconditionally.
    ensureStringBlocksMethod.invoke(newAssetManager);

    for (WeakReference<Resources> wr : references) {
        Resources resources = wr.get();
        //pre-N
        if (resources != null) {
            // Set the AssetManager of the Resources instance to our brand new one
            try {
                assetsFiled.set(resources, newAssetManager);
            } catch (Throwable ignore) {
                // N
                Object resourceImpl = resourcesImplFiled.get(resources);
                // for Huawei HwResourcesImpl
                Field implAssets = ShareReflectUtil.findField(resourceImpl, "mAssets");
                implAssets.setAccessible(true);
                implAssets.set(resourceImpl, newAssetManager);
            }

            resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
        }
    }

    // 使用我们的测试资源文件测试是否更新成功
    if (!checkResUpdate(context)) {
        throw new TinkerRuntimeException(ShareConstants.CHECK_RES_INSTALL_FAIL);
    }
}
```

这里需要提一下资源动态替换的原理，可以参考[文档](http://blog.csdn.net/cauchyweierstrass/article/details/51067729)。主要原理是通过AssertManager的addAssetPath函数，加入外部的资源路径，然后将Resources的mAssets的字段设为前面的AssertManager，这样在通过getResources去获取资源的时候就可以获取到我们外部的资源了。

