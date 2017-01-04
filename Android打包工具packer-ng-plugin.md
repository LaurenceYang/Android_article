# Android打包工具packer-ng-plugin

标签（空格分隔）： Android 开源项目 打包

---

>最近集成Tinker遇到多渠道打包问题，通过我们的打包平台或AS中多FLAVOR打包，因为需要修改渠道号，都会造成DEX的差异。下发补丁时，1个版本100个渠道就会需要100个补丁，非常不方便。Tinker针对这个问题建议将渠道信息写在apk文件的zip comment中。下面就介绍实现了该功能的开源项目[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)。

##原理
Android应用使用的APK文件就是一个带签名信息的ZIP文件，根据 [ZIP文件格式规范](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)，每个ZIP文件的最后都必须有一个叫 [Central Directory Record](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html) 的部分，这个CDR的最后部分叫"end of central directory record"，这一部分包含一些元数据，它的末尾是ZIP文件的注释。注释包含Comment Length和File Comment两个字段，前者表示注释内容的长度，后者是注释的内容，正确修改这一部分不会对ZIP文件造成破坏，利用这个字段，我们可以添加一些自定义的数据，PackerNg项目就是在这里添加和读取渠道信息。

##Talk is cheap.Show me the code.
###向APK文件添加渠道信息
```java
public static void writeZipComment(File file, String comment) throws IOException {
    if (hasZipCommentMagic(file)) {
        throw new MarketExistsException("Zip comment already exists, ignore.");
    }
    // {@see java.util.zip.ZipOutputStream.writeEND}
    byte[] data = comment.getBytes(UTF_8);
    final RandomAccessFile raf = new RandomAccessFile(file, "rw");
    raf.seek(file.length() - SHORT_LENGTH);
    // write zip comment length
    // (content field length + length field length + magic field length)
    writeShort(data.length + SHORT_LENGTH + MAGIC.length, raf);
    // write content
    writeBytes(data, raf);
    // write content length
    writeShort(data.length, raf);
    // write magic bytes
    writeBytes(MAGIC, raf);
    raf.close();
}
```

###从APK文件中读取渠道信息
```java
public static String readZipComment(File file) throws IOException {
    RandomAccessFile raf = null;
    try {
        raf = new RandomAccessFile(file, "r");
        long index = raf.length();
        byte[] buffer = new byte[MAGIC.length];
        index -= MAGIC.length;
        // read magic bytes
        raf.seek(index);
        raf.readFully(buffer);
        // if magic bytes matched
        if (isMagicMatched(buffer)) {
            index -= SHORT_LENGTH;
            raf.seek(index);
            // read content length field
            int length = readShort(raf);
            if (length > 0) {
                index -= length;
                raf.seek(index);
                // read content bytes
                byte[] bytesComment = new byte[length];
                raf.readFully(bytesComment);
                return new String(bytesComment, UTF_8);
            } else {
                throw new MarketNotFoundException("Zip comment content not found");
            }
        } else {
            throw new MarketNotFoundException("Zip comment magic bytes not found");
        }
    } finally {
        if (raf != null) {
            raf.close();
        }
    }
}
```
上面两段为核心代码，实现不是很复杂，所涉及到的格式为：
```java
|comment_length|             comment            |
               |text_area|text_area_length|MAGIC|
```

从代码可以看出，打包过程不涉及代码的重新编译，仅对文件末尾数据进行操作，速度比多Flavor快很多。是非常巧妙的一个idea。赞~
不过有一个缺点：没有使用Android的productFlavors，无法利用flavors条件编译的功能。
更多的信息大家可以到packer-ng-plugin的github里去查看，里面对使用，原理和遇到的坑都有比较全面的描述。

>随带说一下Tinker热更新在集成过程中的另外一个坑：360加固
我们的应用在上传360渠道的时候都要求加固，但加固后就不能进行热更新，而且第三方后台暂时没有针对渠道的下发屏蔽策略。目前考虑的解决方案，修改360加固的包的TinkerId，不让其更新补丁。这样一个版本多个渠道也仅需要下发一个补丁包即可。