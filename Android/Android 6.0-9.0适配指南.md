# Android 6.0-9.0适配指南

> **前言**
>
> 随着Android版本的不断迭代，不同版本间的行为变更也越来越多，这就意味着开发者需要针对不同Android版本来进行适配。本文主要总结一下Android 6.0-9.0的行为变更，并列举出相应的适配方案。

## 目录

- [什么情况下需要考虑适配](#什么情况下需要考虑适配)
- [Android版本的行为变更](#android版本的行为变更)
  * [Android 6.0](#android-6.0)
    + [运行时权限](#运行时权限)
    + [悬浮窗权限](#悬浮窗权限)
  * [Android 7.0](#android-7.0)
    + [应用间共享文件](#应用间共享文件)
    + [APK signature scheme v2签名方案](#apk-signature-scheme-v2签名方案)
  * [Android 8.0](#android-8.0)
    + [权限申请的变化](#权限申请的变化)
    + [应用图标适配](#应用图标适配)
    + [通知适配](#通知适配)
    + [未知来源应用安装](#未知来源应用安装)
    + [静态注册广播无法接收](#静态注册广播无法接收)
    + [后台服务限制](#后台服务限制)
    + [Only fullscreen opaque activities can request orientation](#only-fullscreen-opaque-activities-can-request-orientation)
  * [Android 9.0](#android-9.0)
    + [使用前台服务需要添加权限](#使用前台服务需要添加权限)
    + [明文请求限制](#明文请求限制)
    + [刘海屏适配](#刘海屏适配)

## 什么情况下需要考虑适配

这就要看项目中配置的targetSdkVersion了，targetSdkVersion的意思是目标SDK版本，表示开发者已针对该版本进行了充分测试并做好了适配，因此我们需要根据targetSdkVersion来进行适配处理。简而言之，如果targetSdkVersion高于某一个SDK版本，就需要关注当前targetSdkVersion及其以下版本的行为变更，进行相应的适配处理。

举个例子，Android 6.0（API Level 23）引入了运行时权限，如果targetSdkVersion小于23，那么就不需要动态申请危险权限，所有在AndroidManifest.xml中配置的权限都会自动获取，与手机的Android版本无关；如果targetSdkVersion>=23，那么就需要对权限申请进行适配，即动态申请需要的危险权限。

## Android版本的行为变更

### Android 6.0

#### 运行时权限

Android 6.0版本引入了运行时权限，对于部分权限不能只在AndroidManifest.xml中配置，还需要在代码中动态申请，我们把这些权限称作危险权限。

* **危险权限都有哪些**

Android 6.0的危险权限一共9组24个（后面的版本有添加）

```xml
<!--CALENDAR-->
<uses-permission android:name="android.permission.READ_CALENDAR"/>
<uses-permission android:name="android.permission.WRITE_CALENDAR"/>
<!--CAMERA-->
<uses-permission android:name="android.permission.CAMERA"/>
<!--CONTACTS-->
<uses-permission android:name="android.permission.READ_CONTACTS"/>
<uses-permission android:name="android.permission.WRITE_CONTACTS"/>
<uses-permission android:name="android.permission.GET_ACCOUNTS"/>
<!--LOCATION-->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<!--MICROPHONE-->
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<!--PHONE-->
<uses-permission android:name="android.permission.READ_PHONE_STATE"/>
<uses-permission android:name="android.permission.CALL_PHONE"/>
<uses-permission android:name="android.permission.READ_CALL_LOG"/>
<uses-permission android:name="android.permission.ADD_VOICEMAIL"/>
<uses-permission android:name="android.permission.WRITE_CALL_LOG"/>
<uses-permission android:name="android.permission.USE_SIP"/>
<uses-permission android:name="android.permission.PROCESS_OUTGOING_CALLS"/>
<!--SENSORS-->
<uses-permission android:name="android.permission.BODY_SENSORS"/>
<!--SMS-->
<uses-permission android:name="android.permission.SEND_SMS"/>
<uses-permission android:name="android.permission.RECEIVE_SMS"/>
<uses-permission android:name="android.permission.READ_SMS"/>
<uses-permission android:name="android.permission.RECEIVE_WAP_PUSH"/>
<uses-permission android:name="android.permission.RECEIVE_MMS"/>
<!--STORAGE-->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

* **权限申请方法**

如果targetSdkVersion小于23，那么我们不需要动态申请这些危险权限；如果targetSdkVersion>=23，需要添加代码来动态申请危险权限。官方提供了API来进行权限的检查和申请：

* `checkSelfPermission()` ：该方法用于检查是否已经具有了相关权限
* `requestPermissions()` ：该方法用于申请相关权限

此外，有一个方法需要注意`shouldShowRequestPermissionRationale()`，该方法表示是否需要向用户解释为何申请该权限，经过测试，只有在用户点击了禁止权限，并且没有勾选“不再询问”时，改方法才会返回true，其他情况下均返回false，下面列举了集中常见情况下该方法的返回值。

- 第一次安装后请求权限前调用：false
- 曾经被拒绝过权限后再调用：true
- 曾经被拒绝过权限且勾选了不再询问后再调用：false
- 总是允许权限后再次调用：false

当所有权限被拒绝并且勾选了不再询问后可以引导用户选择跳转到应用设置页面手动授权相应权限，判断是否勾选了“不再询问”可以通过在权限申请回调中调用shouldShowRequestPermissionRationale()方法，如果权限被禁止并且shouldShowRequestPermissionRationale()方法返回为false，则勾选了“不再询问”。跳转设置页面的代码如下：

```java
Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
intent.setData(Uri.fromParts("package", context.getPackageName(), null));
startActivity(intent);
```

完整代码如下：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_androidm_adapte);

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        List<String> requestPermissionList = new ArrayList<>();
        // 检查权限
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
            requestPermissionList.add(Manifest.permission.READ_EXTERNAL_STORAGE);
        }
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
                != PackageManager.PERMISSION_GRANTED) {
            requestPermissionList.add(Manifest.permission.WRITE_EXTERNAL_STORAGE);
        }
        if (requestPermissionList.size() > 0) {
            // 申请权限
            ActivityCompat.requestPermissions(this, requestPermissionList.toArray(new String[requestPermissionList.size()]), 1);
        } else {
            // 所有权限都已允许
        }
    }
}

@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    switch (requestCode) {
        case 1:
            if (grantResults.length > 0) {
                List<String> deniedPermissions = new ArrayList<>();
                for (int i = 0; i < grantResults.length; i++) {
                    if (grantResults[i] != PackageManager.PERMISSION_GRANTED) {
                        // 权限被拒绝
                        deniedPermissions.add(permissions[i]);
                    }
                }
                if (deniedPermissions.size() > 0) {
                    boolean allNeverAskAgain = true;
                    for (int i = 0; i < deniedPermissions.size(); i++) {
                        if (ActivityCompat.shouldShowRequestPermissionRationale(this, deniedPermissions.get(i))) {
                            // 没有勾选不再提示
                            allNeverAskAgain = false;
                            break;
                        }
                    }
                    // 所有的权限都被勾上不再询问时，跳转到应用设置界面，引导用户手动打开权限
                    if (allNeverAskAgain) {
                        Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
                        intent.setData(Uri.fromParts("package", getPackageName(), null));
                        startActivity(intent);
                    }
                } else {
                    // 所有权限都已允许

                }
            }
            break;
        default:
            break;
    }
}
```

还有一点需要注意，**在Android 8.0 之前的版本，同一权限组的任何一个权限被授予了，组内的其他权限也自动被授予，但是Android 8.0之后的版本，需要更明确指定所使用的权限，并且系统只会授予申请的权限，不会授予没有组内的其他权限**，这意味着，如果只申请了外部存储空间读取权限，在低版本下（API < 26）对外部存储空间使用写入操作是没有问题的，但是在高版本（API >= 26）下是会出现问题的，解决方案是需要两个将读和写的权限一起申请。

可以看出，权限申请流程还是有很多重复代码的，可以自己封装一下，或是使用第三方库。

推荐一个第三方权限申请库[XXPermissions](https://github.com/getActivity/XXPermissions)

#### 悬浮窗权限

targetSdkVersion>=23时，使用悬浮窗功能需要添加权限**android.permission.SYSTEM_ALERT_WINDOW**，该权限比较特殊，不属于危险权限，因此不需要进行动态申请，要用户手动开启才能获得。获取该权限步骤如下：

1）在AndroidManife.xml文件中添加权限

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```

2）判断是否开启了悬浮窗权限

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    // 是否开启了悬浮窗权限
    boolean hasOverlayPermission = Settings.canDrawOverlays(this);
    if (hasOverlayPermission) {
        // 开启悬浮窗口
    } else {
        // 跳转至“显示在其他应用的上层”权限界面，引导用户开启权限
        Uri selfPackageUri = Uri.parse("package:" + getPackageName());
        Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, selfPackageUri);
        startActivityForResult(intent, 1);
    }
} else {
    // 开启悬浮窗口
}

@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    switch (requestCode) {
        case 1:
            if (resultCode == RESULT_OK) {
                // 再次判断权限是否已获取
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                    boolean hasOverlayPermission = Settings.canDrawOverlays(this);
                    if (hasOverlayPermission) {
                        // 开启悬浮窗口
                    }
                }
            }
            break;
        default:
            break;
    }
}
```

通过`Settings.canDrawOverlays(context)` 方法判断是否开启了悬浮窗权限，返回true表示已经开启，返回false表示未开启，需要跳转到应用的设置页面，引导用户手动开启该权限，在`onActivityResult()`中接收权限申请结果。该权限的申请与Android 8.0中引入的未知来源应用安装权限很类似，后面也会提到。

<img src="https://ws3.sinaimg.cn/large/005BYqpggy1g2bhime0yfj30u01t0wi7.jpg" width="400" align=center />

### Android 7.0

#### 应用间共享文件

从Android 7.0开始，不允许在App间使用`file://URI` 的方式传递一个File，否则会抛出FileUriExposedException的错误，直接引发Crash。官方提供的解决方案是使用**FileProvider**，通过`content://` 来替换`file://` 。FileProvider是ContentProvider的子类，用于应用间共享文件，当targetSdkVersion>=24时，就需要引入FileProvider来解决应用间共享文件问题。

* **什么场景下需要使用FileProvider**

需要在应用间传递文件的uri时就要使用FileProvider了，使用场景主要有以下三个：

1）调用系统相机拍照

2）裁剪图片

3）调用系统安装器去安装apk

* **如何使用FileProvider**

1）在AndroidManifest.xml文件中配置

```xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="${applicationId}.fileProvider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

可以看到，provider 标签下配置了几个属性：

- name ：配置当前FileProvider的实现类，是固定的的值。
- authorities：配置一个FileProvider的名字，它在当前系统内需要是唯一值，一般命名方法是包名(applicationId)+fileProvider，**需要注意，要保证该属性值在系统中是唯一的，如果两个app中定义了相同的值，则后者无法安装到手机中**。
- exported：表示该FileProvider是否需要公开出去，必须为false。
- granUriPermissions：是否允许授权文件的临时访问权限，必须为true。

2）指定可分享的文件路径

注意到上面的配置中添加了meta-data，里面指向一个xml文件，下一步就要编写这个xml文件。在资源文件res下新建xml目录，新建文件**file_paths.xml**， 文件名不是固定的，但需要与AndroidManifest中的配置保持一致。

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path
        name="external_storage_root"
        path="." />
</paths>
```

paths标签内，必须配置最少一个xxx-path标签，可使用的子节点有以下几个：

**root-path** ：代表设备的根目录new File("/")
**files-path**：代表context.getFilesDir()
**cache-path**：代表context.getCacheDir()
**external-path**：代表Environment.getExternalStorageDirectory()
**external-files-path**：代表context.getExternalFilesDirs()

**external-cache-path**：代表context.getExternalCacheDirs()

每个节点都支持两个属性：**name**和**path** ，比如

```
<external-path
	name="external"
    path="pics" />
```

代表的目录即为：`Environment.getExternalStorageDirectory()/pics`，声明以后，代码就可以使用所声明的文件夹以及其子文件夹了。上面的示例中声明的即是SD卡根目录。

3）使用FileProvider将file转化为`content://uri`

配置完成后，就可以使用FileProvider将file转化为`content://uri` 了，需要使用`FileProvider.getUriForFile()` 方法。该方法有三个参数，分别是context、authority和file，authority需要与在AndroidManifest.xml中配置的 `android:authorities` 一致。调用此方法得到`content://` 形式的一个Uri对象，就可以直接使用了。

4）授予临时读写权限

得到一个 `content://` 的Uri对象之后，其实也无法对其直接使用，还需要对接收这个Uri的应用赋予对应的权限。

权限类型的常量有两种，分别是读和写的权限：

```
public static final int FLAG_GRANT_READ_URI_PERMISSION = 0x00000001;
public static final int FLAG_GRANT_WRITE_URI_PERMISSION = 0x00000002;
```

系统提供了两种授权的方式。

1.使用`context.grantUriPermission(String toPackage, Uri uri, int modeFlags)` 方法

第一个参数是包名，也就是要给哪个应用授权；第二个参数是授予权限的Uri对象；第三个参数就是上面提到的读写权限。

```java
grantUriPermission(getPackageName(), imageUri,
        Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
```

在不需要的时候可以通过`context.revokeUriPermission()` 方法移除权限。

2.使用Intent.addFlags() 进行授权，该方式主要用于针对intent.setData、setDataAndType以及setClipData相关方式传递uri的。

```java
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
```

下面来看一下FileProvider在具体场景下的应用。

**1.调用系统相机拍照**

```java
Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
File file = new File(Environment.getExternalStorageDirectory(), "image.jpg");
Uri imageUri;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
  imageUri = FileProvider.getUriForFile(this, "com.zk.example.fileProvider", file);
} else {
  imageUri = Uri.fromFile(file);
}
intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
startActivityForResult(intent, 1);
```

这里需要提一下，为什么拍照可以不需要进行授权操作呢，这是因为Intent的action为`ACTION_IMAGE_CAPTURE`，在API21以后，当我们调用startActivity，方法内部会将传入的EXTRA_OUTPUT转化为setClipData，并直接添加WRITE和READ权限。

**2.裁剪图片**

```java
File file = new File(Environment.getExternalStorageDirectory(), "image.jpg");
File resultFile = new File(Environment.getExternalStorageDirectory(), "image_crop.jpg");
// 原始图片Uri
Uri oriUri;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    oriUri = FileProvider.getUriForFile(this,"com.zk.example.fileProvider", file);
} else {
    oriUri = Uri.fromFile(file);
}
// 裁剪图片Uri
Uri resultUri = Uri.fromFile(resultFile);
Intent intent = new Intent("com.android.camera.action.CROP");
intent.setDataAndType(oriUri, "image/*");
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION |
        Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
intent.putExtra("scale", true);
// 设置裁剪区域的宽高比例
intent.putExtra("aspectX", 1);
intent.putExtra("aspectY", 1);
// 设置裁剪区域的宽度和高度
intent.putExtra("outputX", 200);
intent.putExtra("outputY", 200);
// 取消人脸识别
intent.putExtra("noFaceDetection", true);
// 图片输出格式
intent.putExtra("outputFormat", Bitmap.CompressFormat.JPEG.toString());
intent.putExtra("return-data", true);
// 裁剪输出文件
intent.putExtra(MediaStore.EXTRA_OUTPUT, resultUri);
startActivityForResult(intent, 1);
```

通过`intent.setDataAndType()` 传递原始图片的uri时需要使用FileProvider进行处理，裁剪输出的图片则不需要，直接使用`Uri.fromFile()` 即可。虽然解决了系统自带的裁剪问题，不过还是推荐项目中使用第三方裁剪框架，比如[uCrop](https://github.com/Yalantis/uCrop)进行图片裁剪操作。

**3.安装apk**

```java
File file = new File(Environment.getExternalStorageDirectory(), "test.apk");
Intent intent = new Intent(Intent.ACTION_VIEW);
Uri fileUri;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    fileUri = FileProvider.getUriForFile(this, "com.zk.example.fileProvider", file);
} else {
    fileUri = Uri.fromFile(file);
}
intent.setDataAndType(fileUri, "application/vnd.android.package-archive");
startActivity(intent);
```

在Android 8.0以上还需要考虑安装未知来源应用的权限，否则无法安装apk，因为不属于FileProvider的内容，这里就不提了，后面还会说到。

#### APK signature scheme v2签名方案

Android 7.0 引入一项新的应用签名方案APK Signature Scheme v2，它能提供更快的应用安装时间和更多针对未授权 APK 文件更改的保护。在默认情况下，Android Studio 2.2和Android Plugin for Gradle 2.2会使用APK Signature Scheme v2和传统签名方案来签署您的应用。

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2b7zpa3fuj30ew09laa3.jpg)

* 只勾选v1签名就是传统方案签署，但是在7.0上不会使用V2安全的验证方式。
* 只勾选V2签名7.0以下会显示未安装，7.0上则会使用了V2安全的验证方式。
* 同时勾选V1和V2则所有版本都没问题。

### Android 8.0

#### 权限申请的变化

* **Android 8.0中PHONE权限组新增了两个权限**

**ANSWER_PHONE_CALLS**： 允许您的应用通过编程方式接听呼入电话。要在您的应用中处理呼入电话，您可以使用 acceptRingingCall() 函数。
**READ_PHONE_NUMBERS **：权限允许您的应用读取设备中存储的电话号码。

* **同一权限组中的权限申请**

在Android 8.0之前，如果应用在运行时请求权限并且被授予该权限，那么系统会将属于同一权限组并且在清单中注册的其他权限也一起授予应用。

对于针对Android 8.0的应用，此行为已被纠正，系统只会授予应用明确请求的权限。但是，一旦用户为应用授予某个权限，则所有后续对该权限组中权限的请求都将被自动批准，不会提示用户。

举个例子，某个应用在其清单中添加了**READ_EXTERNAL_STORAGE**和**WRITE_EXTERNAL_STORAGE**两个权限，当应用申请了**READ_EXTERNAL_STORAGE**权限，并且用户允许了该权限后，如果该应用的targetSdkVersion低于26，那么系统还会同时授予**WRITE_EXTERNAL_STORAGE**权限，因为该权限也属于同一个权限组并且在清单中注册过；如果targetSdkVersion>=26，则系统此时仅会授予**READ_EXTERNAL_STORAGE**，不过，如果该应用后来又申请了**WRITE_EXTERNAL_STORAGE**权限，则系统会立即授予该权限，不会提示用户。

#### 应用图标适配

Android 8.0提出了应用图标的适配规范，如果targetSdkVersion>=26，但是没有适配应用图标，有可能会出现下面这种效果。

![](https://img-blog.csdn.net/20180306210358633)

适配方法很简单，可参考[Android应用图标微技巧，8.0系统中应用图标的适配](https://blog.csdn.net/guolin_blog/article/details/79417483)这篇文章，这里就不提了。

#### 通知适配

Android 8.0为了更好地管理通知提醒，防止不重要的通知打扰用户，新增了通知渠道，用户可以根据渠道来屏蔽一些不想要接收的通知。

* **什么情况需要适配**

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2bc7qsmtfj30hm04lgmb.jpg)

Android 8.0创建通知的方法新增了一个参数channelId，可以看到原来一个参数的方法已经被标记为废弃。当targetSdkVersion>=26时，如果我们没有传入通知渠道参数，那么通知就不会显示。由于目前大多数应用市场都要求targetSdkVersion高于26，因此适配几乎就是强制的。

* **适配方法**

1）创建通知渠道

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    // 创建通知渠道
    NotificationChannel channel = new NotificationChannel("chat", "聊天消息", NotificationManager.IMPORTANCE_DEFAULT);
    NotificationManager notificationManager = (NotificationManager) getSystemService(
            NOTIFICATION_SERVICE);
    notificationManager.createNotificationChannel(channel);
}
```

创建通知渠道时需要传入三个参数，分别是通知渠道id、通知渠道名称和重要等级，其中渠道id可以随便定义，只要保证全局唯一性就可以。渠道名称是给用户看的，需要能够表达清楚这个渠道的用途。重要等级的不同则会决定通知的不同行为，当然这里只是初始状态下的重要等级，用户可以随时手动更改某个渠道的重要等级，App是无法干预的。需要注意这几个API都是Android 8.0以上才有的，要添加判断。

2）创建通知时传入通知渠道

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    Notification notification = new NotificationCompat.Builder(this, "chat")
            .setContentTitle("标题")
            .setContentText("内容")
            .setSmallIcon(R.mipmap.ic_launcher)
            .build();
    manager.notify(1, notification);
} else {
    Notification notification = new NotificationCompat.Builder(this)
            .setContentTitle("标题")
            .setContentText("内容")
            .setSmallIcon(R.mipmap.ic_launcher)
            .build();
    manager.notify(1, notification);
}
```

这样通知就可以正常显示了。

* **管理通知渠道**

<img src="https://ws3.sinaimg.cn/large/005BYqpggy1g2bd5yr8kyj30u01t041g.jpg" width="400" align=center />

用户可以在设置中选择关闭相应渠道的通知，这也会导致一个问题，如果用户关掉了某个重要的渠道通知，那么之后该渠道的通知就不会再显示了，可能会导致一些重要信息无法接收到。解决方法是判断用户是否关闭了渠道通知，如果关闭了就跳转到应用设置页面提示用户手动打开。

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
    NotificationChannel channel = manager.getNotificationChannel("chat");
    // 判断用户是否关闭了渠道通知
    if (channel.getImportance() == NotificationManager.IMPORTANCE_NONE) {
        Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
        intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
        intent.putExtra(Settings.EXTRA_CHANNEL_ID, channel.getId());
        startActivity(intent);
        Toast.makeText(AndroidOAdapteActivity.this, "请手动将通知打开", Toast.LENGTH_SHORT).show();
    }
}
```

此外，系统还提供了删除通知渠道的方法，调用NotificationManager的`deleteNotificationChannel(channelId)` 方法即可删除通知渠道，不过在通知设置界面会显示所有被删除的通知渠道数量，不是很美观，因此不推荐在程序中删除通知渠道。

#### 未知来源应用安装

Android 8.0新增了未知来源安装的权限**android.permission.REQUEST_INSTALL_PACKAGES** ，如果targetSdkVersion>=26时，需要用户手动开启该权限才能安装apk。解决方案如下：

1）在AndroidMainfest.xml文件中添加权限

```xml
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
```

2）通过PackageManager的`canRequestPackageInstalls()` 方法来检查是否已经开启了未知来源安装权限

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_androidn_adapte);

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        boolean hasInstallPermission = getPackageManager().canRequestPackageInstalls();
        if (hasInstallPermission) {
            // 安装应用
        } else {
            // 跳转至“安装未知应用”权限界面，引导用户开启权限
            Uri selfPackageUri = Uri.parse("package:" + getPackageName());
            Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES, selfPackageUri);
            startActivityForResult(intent, 2);
        }
    }
}

@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    switch (requestCode) {
        case 1:
            if (resultCode == RESULT_OK) {
              	// 再次判断权限是否已获取
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    if (getPackageManager().canRequestPackageInstalls()) {
                        // 安装应用
                    }
                }
            }
            break;
        default:
            break;
    }
}
```

`canRequestPackageInstalls()` 方法返回true表示获取了权限，返回false表示没有获取权限，需要跳转至应用的设置页面引导用户开启该权限，在`onActivityResult()`中接收权限申请结果。在实际开发中，更好的做法是弹出一个对话框引导用户手动去开启权限，这里只是为了简单说明。

<img src="https://ws3.sinaimg.cn/large/005BYqpggy1g2b84944qfj30u01t0ae9.jpg" width="400" align=center />

该权限的获取方式比较特殊，和Android 6.0中的悬浮窗权限类似，因为其不属于危险权限，因此并不是通过动态申请权限的方式来获得权限的。

#### 静态注册广播无法接收

Android 8.0出于节省电量，提升续航等方面的考虑，大多数隐式广播通过静态注册的方式无法接收。只有以下隐式广播可以通过静态注册来接收：

```
// Android 8.0上不限制的隐式广播
/**
 * 开机广播
 * Intent.ACTION_LOCKED_BOOT_COMPLETED
 * Intent.ACTION_BOOT_COMPLETED
 */
保留原因：这些广播只在首次启动时发送一次，并且许多应用都需要接收此广播以便进行作业、闹铃等事项的安排。

/**
 * 增删用户
 * Intent.ACTION_USER_INITIALIZE
 * "android.intent.action.USER_ADDED"
 * "android.intent.action.USER_REMOVED"
 */
保留原因：这些广播只有拥有特定系统权限的app才能监听，因此大多数正常应用都无法接收它们。
    
/**
 * 时区、ALARM变化
 * "android.intent.action.TIME_SET"
 * Intent.ACTION_TIMEZONE_CHANGED
 * AlarmManager.ACTION_NEXT_ALARM_CLOCK_CHANGED
 */
保留原因：时钟应用可能需要接收这些广播，以便在时间或时区变化时更新闹铃。

/**
 * 语言区域变化
 * Intent.ACTION_LOCALE_CHANGED
 */
保留原因：只在语言区域发生变化时发送，并不频繁。 应用可能需要在语言区域发生变化时更新其数据。

/**
 * Usb相关
 * UsbManager.ACTION_USB_ACCESSORY_ATTACHED
 * UsbManager.ACTION_USB_ACCESSORY_DETACHED
 * UsbManager.ACTION_USB_DEVICE_ATTACHED
 * UsbManager.ACTION_USB_DEVICE_DETACHED
 */
保留原因：如果应用需要了解这些 USB 相关事件的信息，目前尚未找到能够替代注册广播的可行方案。

/**
 * 蓝牙状态相关
 * BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED
 * BluetoothA2dp.ACTION_CONNECTION_STATE_CHANGED
 * BluetoothDevice.ACTION_ACL_CONNECTED
 * BluetoothDevice.ACTION_ACL_DISCONNECTED
 */
保留原因：应用接收这些蓝牙事件的广播时不太可能会影响用户体验。

}
/**
 * Telephony相关
 * CarrierConfigManager.ACTION_CARRIER_CONFIG_CHANGED
 * TelephonyIntents.ACTION_*_SUBSCRIPTION_CHANGED
 * TelephonyIntents.SECRET_CODE_ACTION
 * TelephonyManager.ACTION_PHONE_STATE_CHANGED
 * TelecomManager.ACTION_PHONE_ACCOUNT_REGISTERED
 * TelecomManager.ACTION_PHONE_ACCOUNT_UNREGISTERED
 */
保留原因：设备制造商 (OEM) 电话应用可能需要接收这些广播。

/**
 * 账号相关
 * AccountManager.LOGIN_ACCOUNTS_CHANGED_ACTION
 */
保留原因：一些应用需要了解登录帐号的变化，以便为新帐号和变化的帐号设置计划操作。

/**
 * 应用数据清除
 * Intent.ACTION_PACKAGE_DATA_CLEARED
 */
保留原因：只在用户显式地从 Settings 清除其数据时发送，因此广播接收器不太可能严重影响用户体验。
    
/**
 * 软件包被移除
 * Intent.ACTION_PACKAGE_FULLY_REMOVED
 */
保留原因：一些应用可能需要在另一软件包被移除时更新其存储的数据；对于这些应用，尚未找到能够替代注册此广播的可行方案。

/**
 * 外拨电话
 * Intent.ACTION_NEW_OUTGOING_CALL
 */
保留原因：执行操作来响应用户打电话行为的应用需要接收此广播。
    
/**
 * 当设备所有者被设置、改变或清除时发出
 * DevicePolicyManager.ACTION_DEVICE_OWNER_CHANGED
 */
保留原因：此广播发送得不是很频繁；一些应用需要接收它，以便知晓设备的安全状态发生了变化。
    
/**
 * 日历相关
 * CalendarContract.ACTION_EVENT_REMINDER
 */
保留原因：由日历provider发送，用于向日历应用发布事件提醒。因为日历provider不清楚日历应用是什么，所以此广播必须是隐式广播。
    
/**
 * 安装或移除存储相关广播
 * Intent.ACTION_MEDIA_MOUNTED
 * Intent.ACTION_MEDIA_CHECKING
 * Intent.ACTION_MEDIA_EJECT
 * Intent.ACTION_MEDIA_UNMOUNTED
 * Intent.ACTION_MEDIA_UNMOUNTABLE
 * Intent.ACTION_MEDIA_REMOVED
 * Intent.ACTION_MEDIA_BAD_REMOVAL
 */
保留原因：这些广播是作为用户与设备进行物理交互的结果：安装或移除存储卷或当启动初始化时（当可用卷被装载）的一部分发送的，因此它们不是很常见，并且通常是在用户的掌控下。

/**
 * 短信、WAP PUSH相关
 * Telephony.Sms.Intents.SMS_RECEIVED_ACTION
 * Telephony.Sms.Intents.WAP_PUSH_RECEIVED_ACTION
 * <p>
 * 注意：需要申请以下权限才可以接收
 * "android.permission.RECEIVE_SMS"
 * "android.permission.RECEIVE_WAP_PUSH"
 */
保留原因：SMS短信应用需要接收这些广播。
```

在AndroidManife.xml中静态注册的广播接收器无法接收到隐式广播，系统会打印

```
BroadcastQueue: Background execution not allowed
```

**解决方案**

1）动态注册广播

```java
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("com.example.adaptetest.MY_BROADCAST");
mReceiver = new MyBroadcastReceiver();
registerReceiver(mReceiver, intentFilter);
```

2）使用显式广播，通过Intent指定目标组件，可以是`intent.setClass()` 、`intent.setComponent()`或`intent.setPackage()`等写法，这样的话静态注册也能接收到广播。

```java
Intent intent = new Intent();
intent.setClass(AndroidOAdapteActivity.this, MyBroadcastReceiver.class);
sendBroadcast(intent);
```

3）将targetSdkVersion改为26以下，不推荐这样做，因为目前很多应用市场要求targetSdkVersion必须高于26。

4）如果还是要发送隐式广播，又不能动态注册，有一种方法可以绕过系统的限制，通过查看上面那段异常信息的来源，最后分析可得，只要发送广播的时候携带`intent.addFlags(0x01000000);` （对应**FLAG_RECEIVER_INCLUDE_BACKGROUND ** 标志位，不过该标志位是找不到的，因此只能通过硬编码来指定） ，就能突破隐式广播的限制。

5）使用JobScheduler，这是一个官方提供的任务调度器，不过我还没有用过。

#### 后台服务限制

Android 8.0出于降低耗电等方面的考虑，限制了后台服务的运行，当应用处于空闲状态时，后台服务运行会受到限制。但是对于前台服务（Foreground Service）则不会有这个限制，因为前台服务对用户是可见的。

系统可以区分**前台**和**后台**应用。如果满足以下任意条件，应用将被视为处于前台：

- 具有可见 Activity（不管该 Activity 已启动还是已暂停）
- 具有前台服务
- 另一个前台应用已关联到该应用（不管是通过绑定到其中一个服务，还是通过使用其中一个内容提供程序）。 例如，如果另一个应用绑定到该应用的服务，那么该应用处于前台：
  - [IME](https://developer.android.google.cn/guide/topics/text/creating-input-method.html)
  - 壁纸服务
  - 通知侦听器
  - 语音或文本服务

如果以上条件均不满足，应用将被视为处于后台。

Android 8.0后台服务限制的具体表现有：

1）当应用进入后台运行一段时间后（在我的手机上测试大约为1分钟），运行着的后台服务会停止，服务的`onDestroy()`方法或被调用，类似调用了服务的`stopSelf()`方法。

2）当应用进入后台运行一段时间后（在我的手机上测试大约为1分钟），再调用`startService()`启动后台服务会抛出以下异常：

**java.lang.IllegalStateException: Not allowed to start service Intent**

**解决方案**

官方提供的解决方案是把后台服务转变为前台服务。Android 8.0提供了一个新的方法`startForegroundService()` 来启动前台服务。

```java
Intent intent = new Intent(AndroidOAdapteActivity.this, MyService.class);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    startForegroundService(intent);
} else {
    startService(intent);
}
```

在Android 9.0上使用前台服务还需要在AndroidManif.xml文件中添加权限，否则会抛出异常：

```xml
<!--Android 9.0上使用前台服务，需要添加权限-->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

**需要注意的是使用这个方法启动的服务必须在5秒内使用`startForeground()`方法来显出示通知，否则将会抛出以下异常：Context.startForegroundService() did not then call Service.startForeground()** ，具体代码如下：

```java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        createNotificationChannel("subscribe", "订阅消息", NotificationManager.IMPORTANCE_DEFAULT);
        Notification notification = new NotificationCompat.Builder(this, "subscribe")
                .setContentTitle("标题")
                .setContentText("内容")
                .setSmallIcon(R.mipmap.ic_launcher)
                .build();
        startForeground(1, notification);
    } else {
        Notification notification = new NotificationCompat.Builder(this)
                .setContentTitle("标题")
                .setContentText("内容")
                .setSmallIcon(R.mipmap.ic_launcher)
                .build();
        startForeground(1, notification);
    }
    return super.onStartCommand(intent, flags, startId);
}

/**
 * 创建通知渠道
 *
 * @param channelId   渠道id
 * @param channelName 渠道名称
 * @param importance  重要等级
 */
@TargetApi(Build.VERSION_CODES.O)
private void createNotificationChannel(String channelId, String channelName, int importance) {
    NotificationChannel channel = new NotificationChannel(channelId, channelName, importance);
    NotificationManager notificationManager = (NotificationManager) getSystemService(
            NOTIFICATION_SERVICE);
    notificationManager.createNotificationChannel(channel);
}
```

创建通知用到了通知的适配，需要传入通知渠道参数。

#### Only fullscreen opaque activities can request orientation

Android 8.0限制非全屏的透明页面不允许设置方向，否则应用会Crash掉，该限制在Android 8.1及以上版本已经修复。解决方案有两种：

1）取消设置页面透明，将页面的**android:windowIsTranslucent**属性设置为false

2）不设置页面的方向

### Android 9.0

#### 使用前台服务需要添加权限

```xml
<!--Android 9.0上使用前台服务，需要添加权限-->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

#### 明文请求限制

Android 9.0限制了明文流量的网络请求（HTTP），非加密的流量请求都会被系统禁止掉。如果targetSdkVersion>=28，使用HTTP请求会在日志中提示以下异常（只是无法正常发出请求，不会导致应用崩溃）。

**java.net.UnknownServiceException: CLEARTEXT communication to xxx not permitted by network security policy**

**解决方案**

1）将网络请求统一改为“https”

2）在资源文件res下新建xml目录，新建文件**network_security_config.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

然后在AndroidManifest.xml中配置：

```xml
<application
    ...
    android:networkSecurityConfig="@xml/network_security_config"
    ...
>
```

3）将targetSdkVersion改为28以下，不推荐

#### 刘海屏适配

* **什么情况下需要适配刘海屏**

1）沉浸式状态栏，窗口布局延伸到了状态栏中，刘海区域可能会遮挡必要的内容或控件，适配方式是将布局下移，预留出状态栏高度。

2）全屏显示模式，不适配的话状态栏会留出一条黑边，布局整体向下移。

* **如何适配**

Android 9.0增加了一个窗口布局参数属性`layoutInDisplayCutoutMode`，该属性有三个值可以取：

>**LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT**：默认的布局模式，仅当刘海区域完全包含在状态栏之中时，才允许窗口延伸到刘海区域显示，也就是说，如果没有设置为全屏显示模式，就允许窗口延伸到刘海区域，否则不允许。
>**LAYOUT_IN_DISPLAY_CUTOUT_MODE_NEVER**：永远不允许窗口延伸到刘海区域。
>**LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES**：始终允许窗口延伸到屏幕短边上的刘海区域，窗口永远不会延伸到屏幕长边上的刘海区域。

使用**LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES** 即可完成全面屏的适配，具体代码如下：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
    WindowManager.LayoutParams lp = getWindow().getAttributes();
    // 始终允许窗口延伸到屏幕短边上的缺口区域
    lp.layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
    getWindow().setAttributes(lp);
} 
```

对于Android 9.0之前的刘海屏适配，需要根据各大厂商的适配方案进行适配。

[华为刘海屏手机安卓O版本适配指导](https://developer.huawei.com/consumer/cn/devservice/doc/50114)

[小米刘海屏水滴屏 Android O 适配](https://dev.mi.com/console/doc/detail?pId=1293)

[Vivo全面屏应用适配指南](https://dev.vivo.com.cn/documentCenter/doc/103)

[Oppo凹形屏适配指南](https://open.oppomobile.com/wiki/doc#id=10159)