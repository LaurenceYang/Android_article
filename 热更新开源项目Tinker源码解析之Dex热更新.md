# 热更新开源项目Tinker源码解析之Dex热更新

标签（空格分隔）： Android 开源项目 热更新

---
Tinker中Dex的热更新主要分为三个部分：一是补丁包的生成；二是补丁包下发后生成全量Dex；三是生成全量Dex后的加载过程。

##一、生成补丁流程
当在命令行里面调用tinkerPatchRelease任务时会调用com.tencent.tinker.build.patch.Runner.tinkerPatch()进行生成补丁生成过程。
```jva
//gen patch
ApkDecoder decoder = new ApkDecoder(config);
decoder.onAllPatchesStart();
decoder.patch(config.mOldApkFile, config.mNewApkFile);
decoder.onAllPatchesEnd();

//gen meta file and version file
PatchInfo info = new PatchInfo(config);
info.gen();

//build patch
PatchBuilder builder = new PatchBuilder(config);
builder.buildPatch();
```

ApkDecoder.patch(File oldFile, File newFile)函数中，会先对manifest文件进行检测，看其是否有更改，如果发现manifest的组件有新增，则抛出异常，因为目前Tinker暂不支持四大组件的新增。检测通过后解压apk文件，遍历新旧apk，交给ApkFilesVisitor进行处理。
```java
//check manifest change first
manifestDecoder.patch(oldFile, newFile);

unzipApkFiles(oldFile, newFile);

Files.walkFileTree(mNewApkDir.toPath(), new ApkFilesVisitor(config, mNewApkDir.toPath(), mOldApkDir.toPath(), dexPatchDecoder, soPatchDecoder, resPatchDecoder));
```

ApkFilesVisitor的visitFile函数中，对于dex类型的文件，调用dexDecoder进行patch操作；对于so类型的文件，使用soDecoder进行patch操作；对于Res类型文件，使用resDecoder进行操作。本文中主要是针对dexDecoder进行分析。
```java
@Override
public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {

    Path relativePath = newApkPath.relativize(file);

    Path oldPath = oldApkPath.resolve(relativePath);

    File oldFile = null;
    //is a new file?!
    if (oldPath.toFile().exists()) {
        oldFile = oldPath.toFile();
    }
    String patternKey = relativePath.toString().replace("\\", "/");

    if (Utils.checkFileInPattern(config.mDexFilePattern, patternKey)) {
        //also treat duplicate file as unchanged
        if (Utils.checkFileInPattern(config.mResFilePattern, patternKey) && oldFile != null) {
            resDuplicateFiles.add(oldFile);
        }

        try {
            dexDecoder.patch(oldFile, file.toFile());
        } catch (Exception e) {
//                    e.printStackTrace();
            throw new RuntimeException(e);
        }
        return FileVisitResult.CONTINUE;
    }
    if (Utils.checkFileInPattern(config.mSoFilePattern, patternKey)) {
        //also treat duplicate file as unchanged
        if (Utils.checkFileInPattern(config.mResFilePattern, patternKey) && oldFile != null) {
            resDuplicateFiles.add(oldFile);
        }
        try {
            soDecoder.patch(oldFile, file.toFile());
        } catch (Exception e) {
//                    e.printStackTrace();
            throw new RuntimeException(e);
        }
        return FileVisitResult.CONTINUE;
    }
    if (Utils.checkFileInPattern(config.mResFilePattern, patternKey)) {
        try {
            resDecoder.patch(oldFile, file.toFile());
        } catch (Exception e) {
//                    e.printStackTrace();
            throw new RuntimeException(e);
        }
        return FileVisitResult.CONTINUE;
    }
    return FileVisitResult.CONTINUE;
```

DexDiffDecoder.patch(final File oldFile, final File newFile)
首先检测输入的dex文件中是否有不允许修改的类被修改了，如loader相关的类是不允许被修改的，这种情况下会抛出异常；如果dex是新增的，直接将该dex拷贝到结果文件；如果dex是修改的，收集增加和删除的class。oldAndNewDexFilePairList将新旧dex对应关系保存起来，用于后面的分析。
```java
excludedClassModifiedChecker.checkIfExcludedClassWasModifiedInNewDex(oldFile, newFile);
...
//new add file
if (oldFile == null || !oldFile.exists() || oldFile.length() == 0) {
    hasDexChanged = true;
    if (!config.mUsePreGeneratedPatchDex) {
        copyNewDexAndLogToDexMeta(newFile, newMd5, dexDiffOut);
        return true;
    }
}
...
// collect current old dex file and corresponding new dex file for further processing.
oldAndNewDexFilePairList.add(new AbstractMap.SimpleEntry<>(oldFile, newFile));
```
UniqueDexDiffDecoder.patch中将新的dex文件加入到addedDexFiles。
```java
public boolean patch(File oldFile, File newFile) throws IOException, TinkerPatchException {
    boolean added = super.patch(oldFile, newFile);
    if (added) {
        String name = newFile.getName();
        if (addedDexFiles.contains(name)) {
            throw new TinkerPatchException("illegal dex name, dex name should be unique, dex:" + name);
        } else {
            addedDexFiles.add(name);
        }
    }
    return added;
}
```

DexDiffDecoder.onAllPatchesEnd()
DexDiffDecoder.generatePatchedDexInfoFile() 
在patch完成后，会调用generatePatchInfoFile生成补丁文件。
DexFiffDecoder.generatePatchInfoFile中首先遍历oldAndNewDexFilePairList，取出新旧文件对。判断新旧文件的MD5是否相等，不相等，说明有变化，会根据新旧文件创建DexPatchGenerator，DexPatchGenerator构造函数中包含了15个Dex区域的比较算法：
>StringDataSectionDiffAlgorithm
TypeIdSectionDiffAlgorithm
ProtoIdSectionDiffAlgorithm
FieldIdSectionDiffAlgorithm
MethodIdSectionDiffAlgorithm
ClassDefSectionDiffAlgorithm
TypeListSectionDiffAlgorithm
AnnotationSetRefListSectionDiffAlgorithm
AnnotationSetSectionDiffAlgorithm
ClassDataSectionDiffAlgorithm
CodeSectionDiffAlgorithm
DebugInfoItemSectionDiffAlgorithm
AnnotationSectionDiffAlgorithm
StaticValueSectionDiffAlgorithm
AnnotationsDirectorySectionDiffAlgorithm


DexDiffDecoder.executeAndSaveTo(OutputStream out) 
这个函数里面会根据上面的15个算法对dex的各个区域进行比较，最后生成dex文件的差异，**这是整个dex diff算法的核心**。
以StringDataSectionDiffAlgorithm为例
>获取oldDex中StringData区域的Item，并进行排序
获取newDex中StringData区域的Item，并进行排序
然后对ITEM依次比较
<0
　说明从老的dex中删除了该String，patchOperationList中添加Del操作
\>0
　说明添加了该String，patchOperationList添加add操作
=0
　说明都有该String， 记录oldIndexToNewIndexMap，oldOffsetToNewOffsetMap
old item已到结尾
　剩下的item说明都是新增项，patchOperationList添加add操作
new item已到结尾
　剩下的item说明都是删除项，patchOperationList添加del操作
最后对对patchOperationList进行优化（
{OP_DEL idx} followed by {OP_ADD the_same_idx newItem} will be replaced by {OP_REPLACE idx newItem}）
关于DexDiff算法，更加详细的介绍可以参考https://www.zybuluo.com/dodola/note/554061，算法名曰二路归并。

对每个区域比较后会将比较的结果写入文件中，文件格式写在DexDataBuffer中
```java
 private void writeResultToStream(OutputStream os) throws IOException {
    DexDataBuffer buffer = new DexDataBuffer();
    buffer.write(DexPatchFile.MAGIC);
    buffer.writeShort(DexPatchFile.CURRENT_VERSION);
    buffer.writeInt(this.patchedDexSize);
    // we will return here to write firstChunkOffset later.
    int posOfFirstChunkOffsetField = buffer.position();
    buffer.writeInt(0);
    buffer.writeInt(this.patchedStringIdsOffset);
    buffer.writeInt(this.patchedTypeIdsOffset);
    buffer.writeInt(this.patchedProtoIdsOffset);
    buffer.writeInt(this.patchedFieldIdsOffset);
    buffer.writeInt(this.patchedMethodIdsOffset);
    buffer.writeInt(this.patchedClassDefsOffset);
    buffer.writeInt(this.patchedMapListOffset);
    buffer.writeInt(this.patchedTypeListsOffset);
    buffer.writeInt(this.patchedAnnotationSetRefListItemsOffset);
    buffer.writeInt(this.patchedAnnotationSetItemsOffset);
    buffer.writeInt(this.patchedClassDataItemsOffset);
    buffer.writeInt(this.patchedCodeItemsOffset);
    buffer.writeInt(this.patchedStringDataItemsOffset);
    buffer.writeInt(this.patchedDebugInfoItemsOffset);
    buffer.writeInt(this.patchedAnnotationItemsOffset);
    buffer.writeInt(this.patchedEncodedArrayItemsOffset);
    buffer.writeInt(this.patchedAnnotationsDirectoryItemsOffset);
    buffer.write(this.oldDex.computeSignature(false));
    int firstChunkOffset = buffer.position();
    buffer.position(posOfFirstChunkOffsetField);
    buffer.writeInt(firstChunkOffset);
    buffer.position(firstChunkOffset);

    writePatchOperations(buffer, this.stringDataSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.typeIdSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.typeListSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.protoIdSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.fieldIdSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.methodIdSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.annotationSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.annotationSetSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.annotationSetRefListSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.annotationsDirectorySectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.debugInfoSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.codeSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.classDataSectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.encodedArraySectionDiffAlg.getPatchOperationList());
    writePatchOperations(buffer, this.classDefSectionDiffAlg.getPatchOperationList());

    byte[] bufferData = buffer.array();
    os.write(bufferData);
    os.flush();
}
```

生成的文件以dex结尾，但不是真正的dex文件，这里要注意一下。


##二、收到补丁后生成全量Dex流程
当app收到服务器下发的补丁后，会触发DefaultPatchListener.onPatchReceived事件，调用TinkerPatchService.runPatchService启动patch进程进行补丁patch工作。

UpgradePatch.tryPatch()中会首先检查补丁的合法性，签名，以及是否安装过补丁，检查通过后会尝试dex，so以及res文件的patch。本文中主要分析DexDiffPatchInternal.tryRecoverDexFiles，讨论dex的patch过程。
```java
DexDiffPatchInternal.tryRecoverDexFiles
BsDiffPatchInternal.tryRecoverLibraryFiles
ResDiffPatchInternal.tryRecoverResourceFiles
rewritePatchInfoFileWithLock
```

tryRecoverDexFiles调用DexDiffPatchInternal.patchDexFile，最终通过DexPatchApplier.executeAndSaveTo进行执行及生产全量dex。
```java
private static void patchDexFile(
        ZipFile baseApk, ZipFile patchPkg, ZipEntry oldDexEntry, ZipEntry patchFileEntry,
        ShareDexDiffPatchInfo patchInfo,  File patchedDexFile) throws IOException {
    InputStream oldDexStream = null;
    InputStream patchFileStream = null;
    try {
        oldDexStream = baseApk.getInputStream(oldDexEntry);
        patchFileStream = (patchFileEntry != null ? patchPkg.getInputStream(patchFileEntry) : null);

        final boolean isRawDexFile = SharePatchFileUtil.isRawDexFile(patchInfo.rawName);
        if (!isRawDexFile || patchInfo.isJarMode) {
            ZipOutputStream zos = null;
            try {
                zos = new ZipOutputStream(new BufferedOutputStream(new FileOutputStream(patchedDexFile)));
                zos.putNextEntry(new ZipEntry(ShareConstants.DEX_IN_JAR));
                // Old dex is not a raw dex file.
                if (!isRawDexFile) {
                    ZipInputStream zis = null;
                    try {
                        zis = new ZipInputStream(oldDexStream);
                        ZipEntry entry;
                        while ((entry = zis.getNextEntry()) != null) {
                            if (ShareConstants.DEX_IN_JAR.equals(entry.getName())) break;
                        }
                        if (entry == null) {
                            throw new TinkerRuntimeException("can't recognize zip dex format file:" + patchedDexFile.getAbsolutePath());
                        }
                        new DexPatchApplier(zis, (int) entry.getSize(), patchFileStream).executeAndSaveTo(zos);
                    } finally {
                        SharePatchFileUtil.closeQuietly(zis);
                    }
                } else {
                    new DexPatchApplier(oldDexStream, (int) oldDexEntry.getSize(), patchFileStream).executeAndSaveTo(zos);
                }
                zos.closeEntry();
            } finally {
                SharePatchFileUtil.closeQuietly(zos);
            }
        } else {
            new DexPatchApplier(oldDexStream, (int) oldDexEntry.getSize(), patchFileStream).executeAndSaveTo(patchedDexFile);
        }
    } finally {
        SharePatchFileUtil.closeQuietly(oldDexStream);
        SharePatchFileUtil.closeQuietly(patchFileStream);
    }
}
```

DexPatchApplier.executeAndSaveTo(OutputStream out)中会对15个dex区域进行patch操作，针对old dex和patch dex进行合并,生成全量dex文件。
```java
 public void executeAndSaveTo(OutputStream out) throws IOException {
    // Before executing, we should check if this patch can be applied to
    // old dex we passed in.
    // 首先old apk的签名和patchfile所携带的old apk签名是否一致，不一致则抛出异常
    byte[] oldDexSign = this.oldDex.computeSignature(false);
    if (oldDexSign == null) {
        throw new IOException("failed to compute old dex's signature.");
    }

    if (this.patchFile != null) {
        byte[] oldDexSignInPatchFile = this.patchFile.getOldDexSignature();
        if (CompareUtils.uArrCompare(oldDexSign, oldDexSignInPatchFile) != 0) {
            throw new IOException(
                    String.format(
                            "old dex signature mismatch! expected: %s, actual: %s",
                            Arrays.toString(oldDexSign),
                            Arrays.toString(oldDexSignInPatchFile)
                    )
            );
        }
    }

    String oldDexSignStr = Hex.toHexString(oldDexSign);

    // Firstly, set sections' offset after patched, sort according to their offset so that
    // the dex lib of aosp can calculate section size.
    // patchedDex是最终合成的dex，首先设定各个区域的偏移量
    TableOfContents patchedToc = this.patchedDex.getTableOfContents();

    patchedToc.header.off = 0;
    patchedToc.header.size = 1;
    patchedToc.mapList.size = 1;

    if (extraInfoFile == null || !extraInfoFile.isAffectedOldDex(this.oldDexSignStr)) {
        patchedToc.stringIds.off
                = this.patchFile.getPatchedStringIdSectionOffset();
        patchedToc.typeIds.off
                = this.patchFile.getPatchedTypeIdSectionOffset();
        patchedToc.typeLists.off
                = this.patchFile.getPatchedTypeListSectionOffset();
        patchedToc.protoIds.off
                = this.patchFile.getPatchedProtoIdSectionOffset();
        patchedToc.fieldIds.off
                = this.patchFile.getPatchedFieldIdSectionOffset();
        patchedToc.methodIds.off
                = this.patchFile.getPatchedMethodIdSectionOffset();
        patchedToc.classDefs.off
                = this.patchFile.getPatchedClassDefSectionOffset();
        patchedToc.mapList.off
                = this.patchFile.getPatchedMapListSectionOffset();
        patchedToc.stringDatas.off
                = this.patchFile.getPatchedStringDataSectionOffset();
        patchedToc.annotations.off
                = this.patchFile.getPatchedAnnotationSectionOffset();
        patchedToc.annotationSets.off
                = this.patchFile.getPatchedAnnotationSetSectionOffset();
        patchedToc.annotationSetRefLists.off
                = this.patchFile.getPatchedAnnotationSetRefListSectionOffset();
        patchedToc.annotationsDirectories.off
                = this.patchFile.getPatchedAnnotationsDirectorySectionOffset();
        patchedToc.encodedArrays.off
                = this.patchFile.getPatchedEncodedArraySectionOffset();
        patchedToc.debugInfos.off
                = this.patchFile.getPatchedDebugInfoSectionOffset();
        patchedToc.codes.off
                = this.patchFile.getPatchedCodeSectionOffset();
        patchedToc.classDatas.off
                = this.patchFile.getPatchedClassDataSectionOffset();
        patchedToc.fileSize
                = this.patchFile.getPatchedDexSize();
    } else {
       ...
    }

    Arrays.sort(patchedToc.sections);

    patchedToc.computeSizesFromOffsets();

    // Secondly, run patch algorithms according to sections' dependencies.
    // 对每个区域进行patch操作
    this.stringDataSectionPatchAlg = new StringDataSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.typeIdSectionPatchAlg = new TypeIdSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.protoIdSectionPatchAlg = new ProtoIdSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.fieldIdSectionPatchAlg = new FieldIdSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.methodIdSectionPatchAlg = new MethodIdSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.classDefSectionPatchAlg = new ClassDefSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.typeListSectionPatchAlg = new TypeListSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.annotationSetRefListSectionPatchAlg = new AnnotationSetRefListSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.annotationSetSectionPatchAlg = new AnnotationSetSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.classDataSectionPatchAlg = new ClassDataSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.codeSectionPatchAlg = new CodeSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.debugInfoSectionPatchAlg = new DebugInfoItemSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.annotationSectionPatchAlg = new AnnotationSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.encodedArraySectionPatchAlg = new StaticValueSectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );
    this.annotationsDirectorySectionPatchAlg = new AnnotationsDirectorySectionPatchAlgorithm(
            patchFile, oldDex, patchedDex, oldToFullPatchedIndexMap,
            patchedToSmallPatchedIndexMap, extraInfoFile
    );

    this.stringDataSectionPatchAlg.execute();
    this.typeIdSectionPatchAlg.execute();
    this.typeListSectionPatchAlg.execute();
    this.protoIdSectionPatchAlg.execute();
    this.fieldIdSectionPatchAlg.execute();
    this.methodIdSectionPatchAlg.execute();
    Runtime.getRuntime().gc();
    this.annotationSectionPatchAlg.execute();
    this.annotationSetSectionPatchAlg.execute();
    this.annotationSetRefListSectionPatchAlg.execute();
    this.annotationsDirectorySectionPatchAlg.execute();
    Runtime.getRuntime().gc();
    this.debugInfoSectionPatchAlg.execute();
    this.codeSectionPatchAlg.execute();
    Runtime.getRuntime().gc();
    this.classDataSectionPatchAlg.execute();
    this.encodedArraySectionPatchAlg.execute();
    this.classDefSectionPatchAlg.execute();
    Runtime.getRuntime().gc();

    // Thirdly, write header, mapList. Calculate and write patched dex's sign and checksum.
    Dex.Section headerOut = this.patchedDex.openSection(patchedToc.header.off);
    patchedToc.writeHeader(headerOut);

    Dex.Section mapListOut = this.patchedDex.openSection(patchedToc.mapList.off);
    patchedToc.writeMap(mapListOut);

    this.patchedDex.writeHashes();

    // Finally, write patched dex to file.
    this.patchedDex.writeTo(out);
}
```

每个区域的合并算法采用二路归并，在old dex的基础上对元素进行删除，增加，替换操作。这里的算法和生成补丁的DexDiff是一个逆向的过程。
```java
private void doFullPatch(
        Dex.Section oldSection,
        int oldItemCount,
        int[] deletedIndices,
        int[] addedIndices,
        int[] replacedIndices
) {
    int deletedItemCount = deletedIndices.length;
    int addedItemCount = addedIndices.length;
    int replacedItemCount = replacedIndices.length;
    int newItemCount = oldItemCount + addedItemCount - deletedItemCount;

    int deletedItemCounter = 0;
    int addActionCursor = 0;
    int replaceActionCursor = 0;

    int oldIndex = 0;
    int patchedIndex = 0;
    while (oldIndex < oldItemCount || patchedIndex < newItemCount) {
        if (addActionCursor < addedItemCount && addedIndices[addActionCursor] == patchedIndex) {
            T addedItem = nextItem(patchFile.getBuffer());
            int patchedOffset = writePatchedItem(addedItem);
            ++addActionCursor;
            ++patchedIndex;
        } else
        if (replaceActionCursor < replacedItemCount && replacedIndices[replaceActionCursor] == patchedIndex) {
            T replacedItem = nextItem(patchFile.getBuffer());
            int patchedOffset = writePatchedItem(replacedItem);
            ++replaceActionCursor;
            ++patchedIndex;
        } else
        if (Arrays.binarySearch(deletedIndices, oldIndex) >= 0) {
            T skippedOldItem = nextItem(oldSection); // skip old item.
            markDeletedIndexOrOffset(
                    oldToFullPatchedIndexMap,
                    oldIndex,
                    getItemOffsetOrIndex(oldIndex, skippedOldItem)
            );
            ++oldIndex;
            ++deletedItemCounter;
        } else
        if (Arrays.binarySearch(replacedIndices, oldIndex) >= 0) {
            T skippedOldItem = nextItem(oldSection); // skip old item.
            markDeletedIndexOrOffset(
                    oldToFullPatchedIndexMap,
                    oldIndex,
                    getItemOffsetOrIndex(oldIndex, skippedOldItem)
            );
            ++oldIndex;
        } else
        if (oldIndex < oldItemCount) {
            T oldItem = adjustItem(this.oldToFullPatchedIndexMap, nextItem(oldSection));

            int patchedOffset = writePatchedItem(oldItem);

            updateIndexOrOffset(
                    this.oldToFullPatchedIndexMap,
                    oldIndex,
                    getItemOffsetOrIndex(oldIndex, oldItem),
                    patchedIndex,
                    patchedOffset
            );

            ++oldIndex;
            ++patchedIndex;
        }
    }

    if (addActionCursor != addedItemCount || deletedItemCounter != deletedItemCount
            || replaceActionCursor != replacedItemCount
    ) {
        throw new IllegalStateException(
                String.format(
                        "bad patch operation sequence. addCounter: %d, addCount: %d, "
                                + "delCounter: %d, delCount: %d, "
                                + "replaceCounter: %d, replaceCount:%d",
                        addActionCursor,
                        addedItemCount,
                        deletedItemCounter,
                        deletedItemCount,
                        replaceActionCursor,
                        replacedItemCount
                )
        );
    }
}
```

在extractDexDiffInternals调用完以后，会调用TinkerParallelDexOptimizer.optimizeAll对生成的全量dex进行optimize操作，会存放在odex目录下。到此，生成Dex过程完成。

##三、加载全量dex流程
TinkerApplication通过反射的方式将实际的app业务隔离，这样可以在热更新的时候修改实际的app内容。另外在TinkerApplication中的onBaseContextAttached中会通过反射调用TinkerLoader的tryLoad加载已经合成的dex。
```java
private static final String TINKER_LOADER_METHOD   = "tryLoad";
private void loadTinker() {
    //disable tinker, not need to install
    if (tinkerFlags == TINKER_DISABLE) {
        return;
    }
    tinkerResultIntent = new Intent();
    try {
        //reflect tinker loader, because loaderClass may be define by user!
        Class<?> tinkerLoadClass = Class.forName(loaderClassName, false, getClassLoader());

        Method loadMethod = tinkerLoadClass.getMethod(TINKER_LOADER_METHOD, TinkerApplication.class, int.class, boolean.class);
        Constructor<?> constructor = tinkerLoadClass.getConstructor();
        tinkerResultIntent = (Intent) loadMethod.invoke(constructor.newInstance(), this, tinkerFlags, tinkerLoadVerifyFlag);
    } catch (Throwable e) {
        //has exception, put exception error code
        ShareIntentUtil.setIntentReturnCode(tinkerResultIntent, ShareConstants.ERROR_LOAD_PATCH_UNKNOWN_EXCEPTION);
        tinkerResultIntent.putExtra(INTENT_PATCH_EXCEPTION, e);
    }
}
```
tryLoadPatchFilesInternal是加载Patch文件的核心函数，主要做了以下的事情：
> * tinkerFlag是否开启，否则不加载
> * tinker目录是否生成，没有则表示没有生成全量的dex，不需要重新加载
> * tinker/patch.info是否存在，否则不加载
> * 读取patch.info，读取失败则不加载
> * 比较patchInfo的新旧版本，都为空则不加载
> * 判断版本号是否为空，为空则不加载
> * 判断patch version directory（//tinker/patch.info/patch-641e634c）是否存在
> * 判断patchVersionDirectoryFile（//tinker/patch.info/patch-641e634c/patch-641e634c.apk）是否存在
> * checkTinkerPackage,（如tinkerId和oldTinkerId不能相等，否则不加载)
> * 检测dex的完整性，包括dex是否全部生产，是否对dex做了优化，优化后的文件是否存在（//tinker/patch.info/patch-641e634c/dex）
> * 同样对so res文件进行完整性检测
> * 尝试超过3次不加载
> * loadTinkerJars/loadTinkerResources/

TinkerDexLoader.loadTinkerJars处理加载dex文件。
```java
// 获取PatchClassLoader 
PathClassLoader classLoader = (PathClassLoader) TinkerDexLoader.class.getClassLoader();

...
// 生产合法文件列表
ArrayList<File> legalFiles = new ArrayList<>();

final boolean isArtPlatForm = ShareTinkerInternals.isVmArt();
for (ShareDexDiffPatchInfo info : dexList) {
    //for dalvik, ignore art support dex
    // dalvik虚拟机中，忽略掉只支持art的dex
    if (isJustArtSupportDex(info)) {
        continue;
    }
    String path = dexPath + info.realName;
    File file = new File(path);

    if (tinkerLoadVerifyFlag) {
        long start = System.currentTimeMillis();
        String checkMd5 = isArtPlatForm ? info.destMd5InArt : info.destMd5InDvm;
        if (!SharePatchFileUtil.verifyDexFileMd5(file, checkMd5)) {
            //it is good to delete the mismatch file
            ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_DEX_MD5_MISMATCH);
            intentResult.putExtra(ShareIntentUtil.INTENT_PATCH_MISMATCH_DEX_PATH,
                file.getAbsolutePath());
            return false;
        }
        Log.i(TAG, "verify dex file:" + file.getPath() + " md5, use time: " + (System.currentTimeMillis() - start));
    }
    legalFiles.add(file);
}

// 如果系统OTA，对这些合法dex进行优化
if (isSystemOTA) {
    parallelOTAResult = true;
    parallelOTAThrowable = null;
    Log.w(TAG, "systemOTA, try parallel oat dexes!!!!!");

    TinkerParallelDexOptimizer.optimizeAll(
        legalFiles, optimizeDir,
        new TinkerParallelDexOptimizer.ResultCallback() {
            @Override
            public void onSuccess(File dexFile, File optimizedDir) {
                // Do nothing.
            }
            @Override
            public void onFailed(File dexFile, File optimizedDir, Throwable thr) {
                parallelOTAResult = false;
                parallelOTAThrowable = thr;
            }
        }
    );
    if (!parallelOTAResult) {
        Log.e(TAG, "parallel oat dexes failed");
        intentResult.putExtra(ShareIntentUtil.INTENT_PATCH_EXCEPTION, parallelOTAThrowable);
        ShareIntentUtil.setIntentReturnCode(intentResult, ShareConstants.ERROR_LOAD_PATCH_VERSION_PARALLEL_DEX_OPT_EXCEPTION);
        return false;
    }
}

// 加载Dex
SystemClassLoaderAdder.installDexes(application, classLoader, optimizeDir, legalFiles);
```
SystemClassLoaderAdder.installDexes中按照安卓的版本对dex进行install，这里应该是借鉴了MultiDex里面的install做法。
```java
public static void installDexes(Application application, PathClassLoader loader, File dexOptDir, List<File> files) throws Throwable {
    if (!files.isEmpty()) {
        ClassLoader classLoader = loader;
        if (Build.VERSION.SDK_INT >= 24) {
            classLoader = AndroidNClassLoader.inject(loader, application);
        }
        //because in dalvik, if inner class is not the same classloader with it wrapper class.
        //it won't fail at dex2opt
        if (Build.VERSION.SDK_INT >= 23) {
            V23.install(classLoader, files, dexOptDir);
        } else if (Build.VERSION.SDK_INT >= 19) {
            V19.install(classLoader, files, dexOptDir);
        } else if (Build.VERSION.SDK_INT >= 14) {
            V14.install(classLoader, files, dexOptDir);
        } else {
            V4.install(classLoader, files, dexOptDir);
        }
        //install done
        sPatchDexCount = files.size();
    
        // Tinker在生成补丁阶段会生成一个test.dex，这个test.dex的作用就是用来验证dex的加载是否成功。test.dex中含有com.tencent.tinker.loader.TinkerTestDexLoad类，该类中包含一个字段isPatch，checkDexInstall就是通过findField该字段判断是否加载成功。
        if (!checkDexInstall(classLoader)) {
            //reset patch dex
            SystemClassLoaderAdder.uninstallPatchDex(classLoader);
            throw new TinkerRuntimeException(ShareConstants.CHECK_DEX_INSTALL_FAIL);
        }
    }
}
```

在讲install具体细节之前，回顾一下具体原理。关于Android的ClassLoader体系，android中加载类一般使用的是PathClassLoader和DexClassLoader
PathClassLoader，源码注释可以看出，android使用这个类作为系统类和应用类的加载器。
```java
/**
 * Provides a simple {@link ClassLoader} implementation that operates on a list
 * of files and directories in the local file system, but does not attempt to
 * load classes from the network. Android uses this class for its system class
 * loader and for its application class loader(s).
 */
```

DexClassLoader，源码注释可以看出，可以用来从.jar和.apk类型的文件内部加载classes.dex文件。
```java
/**
 * A class loader that loads classes from {@code .jar} and {@code .apk} files
 * containing a {@code classes.dex} entry. This can be used to execute code not
 * installed as part of an application.
 *
 * <p>This class loader requires an application-private, writable directory to
 * cache optimized classes. Use {@code Context.getDir(String, int)} to create
 * such a directory: <pre>   {@code
 *   File dexOutputDir = context.getDir("dex", 0);
 * }</pre>
 *
 * <p><strong>Do not cache optimized classes on external storage.</strong>
 * External storage does not provide access controls necessary to protect your
 * application from code injection attacks.
 */
```

ok，到这里，大家只需要明白，Android使用PathClassLoader作为其类加载器，DexClassLoader可以从.jar和.apk类型的文件内部加载classes.dex文件就好了。

PathClassLoader和DexClassLoader都继承自BaseDexClassLoader。在BaseDexClassLoader中有如下源码：
```java
##BaseDexClassLoader.java##
/** structured lists of path elements */
private final DexPathList pathList;

@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    Class clazz = pathList.findClass(name);
    if (clazz == null) {
        throw new ClassNotFoundException(name);
    }
    return clazz;
}

##DexPathList.java##
/** list of dex/resource (class path) elements */
private final Element[] dexElements;
public Class findClass(String name) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    return null;
}

##DexFile.java##
public Class loadClassBinaryName(String name, ClassLoader loader) {
    return defineClass(name, loader, mCookie);
}
private native static Class defineClass(String name, ClassLoader loader, int cookie);
```

通俗点讲：
>一个ClassLoader可以包含多个dex文件，每个dex文件是一个Element，多个dex文件排列成一个有序的数组dexElements，当找类的时候，会按顺序遍历dex文件，然后从当前遍历的dex文件中找类，如果找类则返回，如果找不到从下一个dex文件继续查找。(来自：安卓App热补丁动态修复技术介绍)

install的做法就是，先获取BaseDexClassLoader的dexPathList对象，然后通过dexPathList的makeDexElements函数将我们要安装的dex转化成Element[]对象，最后将其和dexPathList的dexElements对象进行合并，就是新的Element[]对象，因为我们添加的dex都被放在dexElements数组的最前面，所以当通过findClass来查找这个类时，就是使用的我们最新的dex里面的类。以V19的install为例：
```java
private static final class V19 {
    private static void install(ClassLoader loader, List<File> additionalClassPathEntries,
                                File optimizedDirectory)
        throws IllegalArgumentException, IllegalAccessException,
        NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
        /* The patched class loader is expected to be a descendant of
         * dalvik.system.BaseDexClassLoader. We modify its
         * dalvik.system.DexPathList pathList field to append additional DEX
         * file entries.
         */
        Field pathListField = ShareReflectUtil.findField(loader, "pathList");
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        ShareReflectUtil.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList,
            new ArrayList<File>(additionalClassPathEntries), optimizedDirectory,
            suppressedExceptions));
        if (suppressedExceptions.size() > 0) {
            for (IOException e : suppressedExceptions) {
                Log.w(TAG, "Exception in makeDexElement", e);
                throw e;
            }
        }
    }
}
```
因为android版本更新较快，不同版本里面的DexPathList等类的函数和字段都有一些变化，这也是我们在install的时候需要对不同版本进行适配的原因。

到此为止，dex的加载流程完成。