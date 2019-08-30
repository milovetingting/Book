# Java Gradle 插件

## 如何应用

```gradle
apply plugin:'java'
```

## Java插件约定的项目结构

```gradle
Project
|--build.gradle
|--src
    |--main
            |--java
            |--resources
    |--test
        |--java
        |--resources
```

main和test是Java插件为我们内置的两个源代码集合，如果想添加自定义的集合，如vip，则可以这样修改：

```gradle
apply plugin:'java'

sourceSets{
    vip{

    }
}
```

添加一个vip源代码集合，然后我们在src目录下添加vip/java,vip/resources目录，就可以分别存放vip相关的源代码和资源文件了。

特殊情况下，我们需要修改java的文件目录，只需要在build.gradle配置对应的目录即可:

```gradle
sourceSets{
    main{
        java{
            srcDir 'src/java'
        }
        resources{
            srcDic 'src/resources'
        }
    }
}
```

## 如何配置很三方依赖

要想使用第三方依赖，需要告诉Gradle如何找到这些依赖

```gradle
repositories{
    mavenCentral()
}
```

以上脚本我们配置了一个Maven中心库，告诉Gradle可以在Maven中心库中搜寻我们依赖的第三方库。我们也可以从jcenter库、ivy库、本地Maven库、自己搭建的Maven私服库等 ，甚至我们本地配置的文件夹也可以作为一个仓库。

```gradle
repositories{
    mavenCentral()
    maven{
        url 'http://www.mavenurl.com'
    }
}
```
有了仓库后，通过配置来告诉Gradle需要依赖什么：

```gradle
dependencies{
    implementation group:'com.squareup.okhttp3',name:'okhttp',version:'3.0.1'
}
```

以上的简写方式，直接把group,name,version去掉，以:分隔：

```gradle
dependencies{
    implementation 'com.squareup.okhttp3:okhttp:3.0.1'
}
```

除了以上这种编译时依赖，Gradle还提供了编译测试用例时的依赖:testImplementation

Java插件可以为不同的源集在编译和运行时指定不同的依赖：

```gradle
dependencies{
    mainImplementation 'com.squareup.okhttp3:okhttp:3.0.1'
    vipImplementation 'com.squareup.okhttp3:okhttp:2.5.0'
}
```

项目依赖:

```gradle
dependencies{
    implementation project(':demo')
}
```

文件依赖:

```gradle
dependencies{
    implementation files('libs/demo.jar','libs/demo2.jar')
}
```

简写方式：

```gradle
dependencies{
    implementation fileTree(dir:'libs',include:'*.jar')
}
```

## 如何构建一个Java项目

常见的任务：

build任务：构建整个项目。

clean任务：删除build目录以及其它构建生成的文件。

assemble任务：不会执行单元测试，只编译和打包。

check任务：只会执行单元测试。

javadoc任务：生成Java格式的doc api文档。

## 源码集合[SourceSet]概念

SourceSet是Java插件用来描述和管理源代码及其资源的一个概念，是一个Java源代码文件和资源文件的集合。

## Java插件添加的任务

详见p65

## Java插件添加的属性

详见p66

## 多项目构建

```gradle
Project
|--app
    |--app.iml
    |--build.gradle
    |--src
|--base
    |--base.iml
    |--build.gradle
    |--src
```
以上是目录结构，app是主项目，base是我们的基础依赖项目。下面在settings.gradle中配置：

```gradle
include ':app'
project(':app').projectDir=new File(rootDir,'chapter6/demo/app')
include ':base'
project(':base').projectDir=new File(rootDir,'chapter6/demo/base')
```

Gradle为我们提供了基于根项目对其所有子项目的通用配置的方法

```gradle
subprojects{
    apply plugin:'java'

    repositories{
        mavenCentral()
    }
}
```

## 如何发布构件

详见p69

## 生成Idea和Eclipse配置

详见p71


