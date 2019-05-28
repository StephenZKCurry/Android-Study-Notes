# Android架构组件-LiveData

## 1.简介

[LiveData](https://developer.android.google.cn/topic/libraries/architecture/livedata)是Android官方推出的Android Jetpack中Architecture组件的一员，是可观察的数据持有类， 与常规Observable不同，LiveData是生命周期感知的，这确保了LiveDate仅在LifecycleOwner处于Active（STARTED/RESUMED）状态下才通知更新数据并且在DESTROYED状态下移除Observer，避免了内存泄漏。

## 2.简单使用

LiveData基于观察者模式，使用起来很简单。



## 3.原理分析



## 4.应用场景



## 5.参考文章

