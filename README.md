# react-native-apk-installer

react native library for apk install

## 安装

requirement RN >= 0.59

```sh
yarn add https://github.com/CrazyAndy/react-native-apk-installer
react-native link react-native-apk-installer
```

## 使用

```tsx
import RNFS from 'react-native-fs';
import ApkInstaller from 'react-native-apk-installer'

    ...
    try {
        var filePath = RNFS.CachesDirectoryPath + '/com.example.app.apk';
        var download = RNFS.downloadFile({
          fromUrl: 'http://example.com/com.example.app.apk',
          toFile: filePath,
          progress: res => {
              console.log((res.bytesWritten / res.contentLength).toFixed(2));
          },
          progressDivider: 1
        });

        download.promise.then(result => {
          if(result.statusCode == 200) {
            console.log(filePath);
            ApkInstaller.install(filePath);
          }
        });
    }
    catch(error) {
        console.warn(error);
    }
    ...
```

## APK安装

使用系统默认的应用安装器进行安装。通过`intent`对象，我们告诉系统需要执行一个打开APK文件的操作，系统就会调用默认的应用安装器去进行安装。

```java
Intent intent = new Intent();
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
intent.setAction(Intent.ACTION_VIEW);
// Android 7.0
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
intent.setDataAndType(apkUri, "application/vnd.android.package-archive");
context.startActivity(intent)
```

Android 8.0 需要申明这个权限

```
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES"/>
```

## 文件共享

这里是坑了我最长时间的地方。关于`FileProvider`模块，没有看到说的比较清楚的...

在Android 7.0之前，你想给另外一个应用传递文件，只需要一个文件路径(`file://`)就好了，其他应用可以通过这路径直接打开你共享的文件。但是Android 7.0后，这个方法不行了，你必须通过`FileProvider`模块才能给其他应用共享文件

Apk文件安装的时候，系统安装器需要访问我们应用提供的Apk文件，也就产生了文件共享问题。因此在传递文件的时候需要处理Android 7.0上的权限问题。

### FileProvider使用

#### 1. AndroidManifest中配置

`FileProvider`实际上是一个`ContentProvider`，所以要使用它，必须先在`AndroidManifest`中声明它

```xml
<application>
    <provider
        android:name=".ApkInstallerFileProvider"
        android:authorities="${applicationId}.apkinstaller_fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/apk_installer_file_paths"/>
    </provider>
</application>
```

`provider`节点下配置了几个属性

* name: 配置当前`FileProvider`的实现类。**由于`provider`的name需要唯一，为了避免跟应用中其他库提供的`FileProvider`冲突，建议自定义一个`FileProvider`派生类，只需要简单继承自`android.support.v4.content.FileProvider`类即可**
* authorities: 配置一个`FileProvider`的名字，它在当前系统内需要是唯一值。
* exported: 表示该`FileProvider`是否需要公开出去，这里不需要，所以是 false。
* granUriPermissions: 是否允许授权文件的临时访问权限。这里需要，所以是 true。

#### 2.指定分享的路径

`meta-data`中的`resource`指定的文件里存了那些路径是支持分享出去的。具体如`cache-path`对应的路径可以在[官方文档](https://developer.android.com/reference/android/support/v4/content/FileProvider#SpecifyFiles)的查看

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-cache-path
        name="external-cache-path"
        path="." />

    <external-media-path 
       name="external-media-path"  
       path="." />

    <cache-path
        name="cache-path"
        path="." />

    <external-path
        name="external-path"
        path="." />

    <files-path
        name="files-path"
        path="." />

    <external-files-path
        name="external-files-path"
        path="." />
</paths>
```

#### 3. 使用

**获取分享路径**

```java
// 获取分享的路径
// 形式如: content://authorites/<分享路径name:如cache-path>/xxx.apk
apkUri = ApkInstallerFileProvider.getUriForFile(context, authorities, apkFile)
```

`authorities`是`AndroidManifest.xml`文件中定义的`authorities`值。

**授权访问**

对于需要接收分享的文件的应用，我们还需要授权它有临时访问我们分享文件的权限。有两种方式来为其他应用授权分享文件访问权限。

```java
grantUriPermission(toPackage, fileUri, modeFlags)
Intent.addFlag()
```

### 4. 注意点！！！

上述配置中`Provider`标签的属性值`name`、`authorities`以及`meta-data`中的`resource`都需要保证唯一，避免被应用内其他依赖库覆盖。

**`name`属性重复的时候，打包时会有提示。但是`authorities`或者`resource`重复了，不会有提示。比如`resoure`指定的文件名有重复，那么可能你指定的分享路径被覆盖了，导致其实分享的路径是其他目录，这样后续`getUriFile`的时候就会在分享路径中找不到对应的文件。**

上面说的这个问题就坑了我许久，如果你也遇到文件找不到的问题，建议使用`apktool d xxx.apk`反编译一下生成包，看看对应的`AndroidManifest.xml`以及`resource`指定文件内容是否正确。


