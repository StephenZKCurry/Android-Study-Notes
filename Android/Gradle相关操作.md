# Android Studio中的Gradle相关操作

> **前言**
>
> Gradle是Android Studio采用的构建工具，给我们的开发提供了诸多便利，但是在使用Gradle的过程中，也有一些需要注意的地方，本文主要记录一下使用Gradle的一些常见问题和技巧。

## 目录

- [1.Gradle插件版本和Gradle版本对应关系](#1gradle插件版本和gradle版本对应关系)
- [2.Gradle下载](#2gradle下载)
- [3.解决support包冲突](#3解决support包冲突)
- [4.依赖统一管理](#4依赖统一管理)

## 1.Gradle插件版本和Gradle版本对应关系

* Gradle插件版本配置

配置在项目的**build.gradle**文件中

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i4z44lnkj30p70b6tc3.jpg)

* Gradle版本配置

配置在项目根目录\gradle\wrapper下的**gradle-wrapper.properties**文件中

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i4yibri3j30ms05yab0.jpg)

* 版本对应关系

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i55vs6mej30my0cxwek.jpg)

[官方链接](https://developer.android.google.cn/studio/releases/gradle-plugin.html#updating-plugin)

## 2.Gradle下载

我们在新建或打开一个项目时需要注意项目的Gradle版本，配置在项目根目录\gradle\wrapper下的**gradle-wrapper.properties**文件中，如果我们本地没有该版本的Gradle，就会从网络下载，由于网络的原因，有时下载得会很慢。我们可以提前将相应版本的Gradle下载到本地，这样就省去了项目在build过程中下载Gradle的时间。

[Gradle下载地址](http://services.gradle.org/distributions/)

## 3.解决support包冲突

我们的项目引入第三方库有时会遇到support包冲突的问题，原因就是第三库也引入了support包，但是和我们自己项目中的support包版本不一致。

首先我们要知道是哪个第三方库导致的冲突，在Android Studio的**Terminal**中运行`gradle -q app:dependencies`命令，会打印出项目中所有的依赖库，包括第三方库自身的依赖。

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2sqqbv321j30qj0dv3z6.jpg)

这里我们可以看出我引入的**PhotoVIew**这个第三方库依赖的support包版本为23.0.0，但是我的项目依赖的support包版本为28.0.0，因此冲突就是这个库导致的。

解决support包版本冲突的方法主要有以下三种：

1）修改自己项目的support包版本，不推荐这样做

2）依赖第三方库时排除对support包的依赖

```groovy
implementation('com.github.chrisbanes:PhotoView:1.2.6') {
    exclude module: 'support-v4'
    exclude group: 'com.android.support'
}
```

当有多个第三方库都引入了support包，需要每个依赖都去排除，这种方法就不太适合了。

3）指定统一的support包版本

在app下的**build.gradle**文件中添加以下代码：

```groovy
configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
        def requested = details.requested
        if (requested.group == 'com.android.support') {
            if (!requested.name.startsWith("multidex")) {
                details.useVersion '28.0.0'
            }
        }
    }
}
```

4）将项目迁移到AndroidX，虽然这是官方的解决方案，但是目前还有一些坑，需要谨慎操作。

## 4.依赖统一管理

在开发过程中，特别是涉及到模块化开发的时候，为了保证全部模块都使用同一个依赖库的管理，一般有两种方法：

1）新建**config.gradle**文件，里面配置第三方库的依赖

* 在项目根目录下新建一个**config.gradle**文件（文件名随意），配置所有的第三方库依赖

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i3wfjlhoj308n0cudfw.jpg)

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i43t63zij30ri0f4tcw.jpg)

* 在Project的**build.gradle**文件中添加依赖

![](https://ws3.sinaimg.cn/large/005BYqpggy1g2i44dxmukj30r20c3tcj.jpg)

* 在相应的子模块中使用

```groovy
android {
    compileSdkVersion rootProject.ext.android.compileSdkVersion
    defaultConfig {
        applicationId "com.zk.wanandroid"
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode rootProject.ext.android.versionCode
        versionName rootProject.ext.android.versionName
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
	...
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.2'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.2'
    // Android support
    implementation rootProject.ext.dependencies["appcompat-v7"]
    implementation rootProject.ext.dependencies["design"]
    implementation rootProject.ext.dependencies["cardview"]
    implementation rootProject.ext.dependencies["constraint-layout"]
    ...
}
```

2）直接在Project的**build.gradle**文件下添加ext{}结点，配置依赖

其实就是相当于将**config.gradle**文件中的内容移到了**build.gradle**中，使用方法同上。
