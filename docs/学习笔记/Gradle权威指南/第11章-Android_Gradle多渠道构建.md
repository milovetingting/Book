# Android Gradle 多渠道构建

## 多渠道构建的基本原理

在Android Gradle中，定义了一个叫Build Variant的概念，一个Build Variant=Build TYpe+Product Flavor，Build Type就是我们构建的类型，比如release和debug;Product Flavor就是我们构建的渠道，比如Baidu,Google等，它们加起来就是baiduRelease,baiduDebug,googleRelease,googleDebug，共有这几种组合的构件产出。Product Flavor也就是我们多渠道构建的基础。以下是新增一个ProductFlavor:

```gradle
android{
    productFlavors{
        google{}
        baidu{}
    }
}
```

以上的发布渠道配置后，Android Gradle就会生成很多Task。其中，assemble开头的负责生成构件产物(apk)。除此之外，还有compile系列，install系列等。除了生成的Task外，每个ProductFlavor还可以有自己的SourceSet，还可以有自己的dependencies依赖。

## Flurry多渠道和友盟多渠道构建

Flurry本身没有渠道的概念，它有Application，所以可以把一个Application当成一个渠道。

```gradle
android{
    productFlavors{
        google{
            buildConfigField 'String','FLURRY_KEY','"ABADFASSDFAS"'
        }
        baidu{
            buildConfigField 'String','FLURRY_KEY','"JKKJKHKJHIHIUY"'
        }
    }
}
```

这样每个渠道的BuildConfig类中都会有名字为FLURRY_KEY的常量定义，它的值是我们在渠道中使用buildConfigField指定的值，每个渠道不一样，我们只需要在代码中指定使用这个常量即可，这样每个渠道的统计分析就可以做到了。

```java
Flurry.init(this,FLURRY_KEY);
```

友盟本身有渠道的概念。不过它不是在代码中指定的，而是在AndroidManifest.xml中配置的，通过配置meta-data标签来设置:

```xml
<meta-data android:name="UMENG_CHANNEL" android:value="Channel ID"/>
```

## 多渠道构建定制

多渠道的定制，其实就是对Android Gradle插件的ProductFlavor的配置，通过配置ProductFlavor达到灵活控制每一个渠道的目的。

### applicaitonId

用于设置渠道的包名

### consumerProguardFiles

只对Android库项目有用。当我们发布库项目生成一个aar包的时候，使用consumerProguardFiles配置的混淆文件列表也会被打包到aar里一起发布，这样当应用引用这个aar包，并且启用混淆的时候，会自动使用aar包里的混淆文件对aar包里的代码进行混淆，这样我们就不用对该aar包进行混淆配置了。

```gradle
android{
    productFlavors{
        google{
            consumeProguardFiles 'proguard-rules.pro','proguard-android.txt'
        }
    }
}
```

除了这种方法，还有一种属性设置的方法，区别在于：consumerProguardFiles方法是一直添加，不会清空以前的混淆文件，而consumerProguardFiles属性配置的方式是每次都是新的混淆文件列表，以前配置的会先被清空。

### manifestPlaceholders

### multiDexEnabled

用来启用多个dex的配置，主要用来突破65535方法的问题

### proguardFiles

混淆使用的文件列表

### signingConfig

签名配置

### testApplicationId

用来适配测试包的包名

### testFunctionalTest和testHandleProfiling

testFunctionalTest表示是否为功能测试，testHandleProfiling表示是否启用分析功能

### testInstrumentationRunner

用来配置运行测试使用的Instrumentation Runner的类名，是一个全路径的类名，而且必须是android.app.Instrumentation的子类，一般情况下，我们使用android.test.InstrumentationTestRunner，当然也可以自定义。

### testInstrumentationRunnerArguments

配合上一个属性用的，用来配置Instrumentation Runner使用的参数，它们最终都是使用adb shell am instrument这个命令。

### versionCode和versionName

配置渠道的版本号和版本名称。

### useJack

用于标记是否启用Jack和Jill这个全新的，高性能的编译器。

### dimension

dimension是ProductFlavor的一个属性，接受一个字符串，作为该ProdoctFlavor的维度。可以简单理解为对ProductFlavor进行分组，dimension接受的参数就是我们分组的组名，也就是维度名称。维度名称不能随便指定，在使用前，必须先声明。

flavorDimension是我们使用的android{}里面的方法，它和productFlavors{}是平级的，一定要先使用flavorDimension声明维度，才能在ProductFlavor中使用。

我们同时指定多个维度，但是一定要，这些维度是有顺序的，有优先级的，第一个参数的优先级最大，其实是第二个，以此类推。

```gradle
android{
    flavorDimensions "abi","version"
}
```

声明维度后，就可以使用了:

```gradle
android{
    flavorDimensions "abi","version"

    productFlavors{
        free{
            dimension 'version'
        }
        paid{
            dimension 'version'
        }
        x86{
            dimension 'abi'
        }
        arm{
            dimension 'abi'
        }
    }
}
```

## 提高多渠道构建的效率

参考美团方案

利用在apk的META-INF目录下添加空文件不用重新签名的原理

1、利用Android Gradle打一个基本包(母包)

2、基于该包复制一个，文件名要能区分出产品、打包时间 、版本、渠道等 

3、对复制出来的apk进行修改，在其META-INF目录下新增空文件，但空文件的文件名要有意义，必须包含能区分渠道的名字

4、重复步骤2、3生成我们所需的所有渠道包apk
