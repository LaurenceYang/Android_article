# Android6.0适配

标签（空格分隔）： Android 6.0适配

---

Android6.0适配主要分为两部分

##一、接口部分适配
###1、网络
Android6.0移除了HttpClient，建议使用OKHTTP或HttpURLConnection替换。
替换时如果HttpClient调用地方较多，如替换到OKHttp时可使用适配器。

如果要保留HttpClient方式，google官网提供了以下解决方案：
build.gradle中加上
```gradle
android {
    useLibrary 'org.apache.http.legacy'
}
```

###2、通知
Android6.0移除了Notification.setLatestEventInfo()方法。
需用Notification.Builder类来构建对象。
还需注意的是，Notification.Builder.build()函数仅在API16及以上版本有效。
API16以下可通过getNotification()创建Notification。
```java
if (Build.VERSION.SDK_INT < 16) {
	notification = builder.getNotification();
} else {
	notification = builder.build();
}
```

###3、FloatMath
Android6.0移除了FloatMath，使用Math替换。

###4、音频音量相关
Android6.0不再支持通过AudioManager类来直接对特定的音频流设置音量和静音。
因此：
setStreamSolo()方法过时（deprecated），替换调用requestAudioFocus()方法；
setStreamMute()方法过时，替换调用为adjustStreamVolume()方法，传入的值也变为ADJUST_MUTE 或 ADJUST_UNMUTE。

###5、文本选择（Text Selection）
用户在应用中选择文字后，你现在可以显示一个浮动工具栏（floatingtoolbar）,展示并进行剪切、拷贝、粘贴操作，交互过程的实现和contextual action bar的实现一样，实现选择文字后的浮动工具栏，在app代码中需要做如下修改：
（1）在View 或 Activity对象，ActionMode的调用从startActionMode(Callback) 变为startActionMode(Callback, ActionMode.TYPE_FLOATING)
（2）替换原有的ActionMode.Callback为ActionMode.Callback2
（3）重写OnGetContentRect()方法，提供内容Rect对象（文本选择的矩形框）在view中的位置
（4）在矩形框作为唯一的元素不再有效时，调用invalidateContentRect()方法
 
如果你在使用Android Support Library revision22.2，需要注意浮动工具栏不向后兼容且因为appcompat默认接管ActionMode对象，阻止了浮动工具栏被显示。为了在 AppCompatActivity中支持ActionMode，需要调用getDelegate()方法，之后对返回的 AppCompatDelegate对象调用setHandleNativeActionModesEnabled()方法，并设置输入参数为 false，该调用将ActionMode对象的控制交还给系统框架层。在Android6.0(API level 23)的设备上，框架层支持ActionBar或浮动工具栏模式，在Android 5.1(API level 22)及以下的设备上，只支持ActionBar模式。

##二、权限部分适配
从Android6.0开始，用户开始在应用运行时向其授予权限，而不是应用安装时授予。
权限分为三类:
###1、正常权限
正常权限涵盖应用需要访问其沙盒外部数据或资源，但对用户隐私或其他应用操作风险很小的区域。例如，设置时区的权限就是正常权限。如果应用声明其需要正常权限，系统会自动向应用授予该权限。
https://developer.android.com/guide/topics/security/normal-permissions.html

###2、危险权限
危险权限涵盖应用需要涉及用户隐私信息的数据或资源，或者可能对用户存储的数据或其他应用的操作产生影响的区域。例如，能够读取用户的联系人属于危险权限。如果应用声明其需要危险权限，则用户必须明确向应用授予该权限。

危险权限的适配分为三步：
**检查权限**
```java
// Assume thisActivity is the current activity
int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,
        Manifest.permission.WRITE_CALENDAR);
```

**请求权限**
```java
// Here, thisActivity is the current activity
if (ContextCompat.checkSelfPermission(thisActivity,
                Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {

        // Show an expanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {

        // No explanation needed, we can request the permission.

        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}
```

**处理权限请求响应**
```java
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
            // If request is cancelled, the result arrays are empty.
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // permission was granted, yay! Do the
                // contacts-related task you need to do.

            } else {

                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return;
        }

        // other 'case' lines to check for other
        // permissions this app might request
    }
}
```

###3、特殊权限
有许多权限其行为方式与正常权限及危险权限都不同。SYSTEM_ALERT_WINDOW 和 WRITE_SETTINGS 特别敏感，因此大多数应用不应该使用它们。如果某应用需要其中一种权限，必须在清单中声明该权限，并且发送请求用户授权的 intent。系统将向用户显示详细管理屏幕，以响应该 intent。

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
	if (!Settings.System.canWrite(this)) {
		Intent intent = new Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS,
				Uri.parse("package:" + getPackageName()));
		startActivity(intent);
	}
}

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
	if (!Settings.canDrawOverlays(mContext)) {
		Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
				Uri.parse("package:" + mContext.getPackageName()));
		startActivity(intent);
	}
}
```


>但对于此套权限模型，不同的手机产商有自己不同的策略。我们发现，Nexus系6.0手机严格遵循着这套模型；但
国内某些手机产商如小米，默认授权了某些危险权限，可能是为了考虑手机升级到6.0后应用市场App的兼容。
适配权限时最重要的是找到权限使用时代码的路径，但存在某些危险权限的代码路径存在第三方lib中，这种情况下只能在首次使用lib前进行权限的请求，但要结合自己的业务需求考虑带来的用户体验问题。
如我们的应用大多在第三方lib中使用了危险权限（读取短信，地址信息等），例如支付流程，如果在这些流程中进行权限的申请，可能造成流程的中断，影响用户体验及业务的需要。因此这种情况我们只适配了特殊权限，然后targetSdkVersion回退到22，让系统进行危险权限的授权。

参考文档：  

http://open.letv.com/guide/?page_id=2400  

https://developer.android.com/training/permissions/requesting.html#explain  




