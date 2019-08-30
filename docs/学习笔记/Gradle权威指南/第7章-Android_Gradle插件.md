# Android Gradle 插件

## Android Gradle 插件简介

从Gradle角度来看，Android其实是Gradle的一个第三方插件，它是由Google的Android团队开发的。但从Android角度 来看，Android插件是基于Gradle构建的，和Android Studio完美搭配的新一代构建系统。

## Android Gradle 插件分类

在Android中有三类工程，一类是App应用工程，它可以生成一个可运行的apk应用。一类是Library库工程，它可以生成AAR包给其它工程使用。一类是Test测试工程，用于对App工程或者Library库工程进行单元测试。

App插件id: com.android.application

Library插件id: com.android.library

Test插件id: com.android.test

## 应用Android Gradle插件

要应用一个插件，必须知道它们的插件id,如果是第三方插件，还需要配置它们的依赖classpath。Android Gradle插件就是第三方插件，它托管在Jcenter上，所在在应用前，需要配置依赖classpath，这样应用插件的时候，Gradle才能找到它们。

```gradle
buildscript{
    repositories{
        jcenter()
    }
    dependencies{
        classpath 'com.android.tools.build:gradle:1.5.0'
    }
}
```

配置好后，就可以应用插件了

```gradle
apply plugin:'com.android.application'

android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"
}
```

## Android Gradle 工程示例

详见p75

Android Gradle工程的配置，都是在android{}中，这是唯一的入口 。

### compileSdkVersion

### buildToolsVersion

### defaultConfig

defaultConfig是默认的配置。它是一个ProductFlavor，ProductFlavor允许我们根据不同情况同时生成多个不同的APK包。

### buildTypes

## Android Gradle 任务

## 从Eclipse迁移到Android Gradle工程

### 使用Android Studio导入

### 从Eclipse+ADT导出
