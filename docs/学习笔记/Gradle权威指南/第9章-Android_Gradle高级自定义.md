# Android Gradle 高级自定义

## 使用共享库

Android的包,如android.app,android.content,android.view,android.widget等，是默认包含在Android SDK库里的，所有应用都可以直接使用它们。还有一些库，如com.google.android.maps,android.test.runner等，这些库是独立的，并不会被系统自动链接，所以如果要使用的话，就需要单独进行生成使用，这类库我们称为共享库。

在AndroidManifest.xml中，我们可以指定要使用的库:

```gradle
<uses-library 
android:name="com.google.android.maps"
android:required="true"
/>
```

这样我们就声明了需要使用maps这个共享库。声明之后，在安装生成的apk包的时候，系统会根据我们的定义，帮助检测手机系统是否有我们需要的共享库。因为我们设置的android:required="true",如果手机系统不满足，将不能安装该应用。

在Android中，除了标准的SDK，还存在两种库：一种是add-ons库，它们位于add-ons目录下，这些库大部分是第三方厂商或者公司开发的，一般是为了开发者使用，但又不想暴露具体标准实现；第二类是optional可选库，它们位于platforms/androi-xx/optional目录下，一般是为了兼容旧版本的API，比如org.apache.http.legacy。

对于第一类add-ons附件库来说，Android Gradle会自动解析，帮我们添加到classpath里。第二类optional可选库就不会，需要自己将这个可选库添加到classpath中。Android Gradle提供了useLibrary方法，让我们把一个库添加到classpath中。

```gradle
android{
    useLibrary 'org.apache.http.legacy'
}
```

以上的配置已经可以生成APK，并能安装运行。但最好也要在AndroidManifest文件中配置一下uses-library标签，以防出现问题。

## 批量修改生成的apk文件名

Android对象为我们提供了3个属性：applicationVariants（仅仅适用于Android应用Gradle插件），libraryVariants（仅仅适用于Android库Grdle插件），testVariants（以上两种Gradle插件都适用）。

以上3个属性返回的都是DomainObjectSet对象集合，访问它们都会触发创建所有的任务。

以下为批量修改apk名称的示例:

```gradle
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.wangyz.gradle"
        minSdkVersion 21
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        flavorDimensions "default"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    productFlavors {
        baidu {

        }
        huawei {

        }
    }
    applicationVariants.all {
        variant ->
            variant.outputs.all {
                output ->
                    if (output.outputFile != null && output.outputFile.name.endsWith('.apk') && 'release'.equals(variant.buildType.name)) {
                        def flavorName = variant.flavorName.startsWith("_") ? variant.flavorName.substring(1) : variant.flavorName
                        def fileName = "channel_${flavorName}_${variant.versionName}.apk"
                        outputFileName = fileName
                    }
            }
    }
}
```

## 动态生成版本信息

一般的版本由3部分组成：major.minor.patch，第一个是主版本号，第二个是副版本号，第三个是补丁号。

### 最原始的方式

```gradle
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.wangyz.channel"
        minSdkVersion 21
        targetSdkVersion 28
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

### 分模块的方式

可以把版本号的配置单独抽取出来，放在单独的文件里，供build引用。

新建一个vesion.gradle文件：

```gradle
ext{
    appVersionCode = 1
    appVersionName = "1.0.0"
}
```

在build.gradle中引用它：

```gradle

apply from:'version.gradle'

android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.wangyz.channel"
        minSdkVersion 21
        targetSdkVersion 28
        versionCode appVersionCode
        versionName appVersionName
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

先使用apply from加载version.gradle脚本文件，这样它里面定义的扩展属性就可以使用了。

### 从git的tag中读取

想获取当前的tag名称，在git下非常简单，使用以下命令即可：

```gradle
git describe --abbrev=0 --tags
```

知道了命令，那么如何在Gradle中动态获取呢？这就需要exec。Gradle提供了执行shell命令非常简便的方法，即exec。它是一个Task任务。

```gradle
def getAppVersionName(){
    def stdout = new ByteArrayOutputStream()
    exec{
        commandLine 'git','describe','--abbrev=0','--tags'
        standardOutput = stdout
    }
    return stdout.toString()
}
```

以上定义了一个获取版本名称的方法，通过该方法获取了git tag的名称后，就可以把它作为应用的版本名称，只要把versionName配置成这个方法就好了。

```gradle
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.wangyz.channel"
        minSdkVersion 21
        targetSdkVersion 28
        versionCode 1
        versionName getAppVersionName()
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}
```

### 从属性文件中动态获取和递增

1、在项目目录下新建一个version.properties的属性文件

2、把版本名称分为3部分major.minor.patch，版本号分为一部分number，然后在properties新增4个KV键值对

3、在build.gradle新建一个方法用于读取该属性文件

## 隐藏签名文件信息

定义一个文件，用来保存签名的相关信息，如sign.properties,这个文件加入.gitignore，不上传到git中。通过读取这个文件来获取配置信息。

```gradle
android{
    signingConfigs{
        release{
            def signPropertiesFile = 'sign.properties'
            storeFile file(readSignProperties(signPropertiesFile,'storeFile'))
            storePassword readSignProperties(signPropertiesFile,'storePassword')
            keyAlias readSignProperties(signPropertiesFile,'keyAlias')
            keyPassword readSignProperties(signPropertiesFile,'keyPassword')
        }
    }
}

buildTypes{
    release{
        signingConfig signingConfigs.release
    }
}

def readSignProperties(String filePath,String key){
    File file = file(filePath)
    if(file.exists())
    {
        def Properties properties = new Properties()
        properties.load(new FileInputStream(file))
        return properties[key]
    }
    return "file not exist!"
}
```

sign.properties内容如下:

```gradle
storeFile=android.keystore
storePassword=android
keyAlias=android
keyPassword=android
```

## 动态配置AndroidManifest文件

Android Gradle 提供了非常便捷的方法让我们来替换AndroidManifest文件中的内容，它就是manifestPlaceholder，Manifest占位符。

```gradle
android{
    productFlavors{
        google{
            manifestPlaceholders = [APP_CHANNEL:"google"]
        }
        baidu{
            manifestPlaceholders = [APP_CHANNEL:"baidu"]
        }
    }
}
```

在AndroidManifest文件中使用

```gradle
<application>
<meta-data 
android:name="APP_CHANNEL"
android:value="${APP_CHANNEL}"
/>
</application>
```

如果需要批量修改(假设需要将名称改为和渠道名一样)，可以通过productFlavors迭代方法:

```gradle
android{
    productFlavors{
        google{

        }
        baidu{

        }
    }

    productFlavors.all{
        flavor->
        manifestPlaceholder = [APP_CHANNEL:name]
    }
}
```

## 自定义BuildConfig

Android Gradle 提供了buildConfigField(String type,String name,String value)让我们添加自己的常量到BuildConfig中。使用示例:

```gradle
android{
    productFlavors{
        google{
            buildConfigField 'String','URL','"http://www.google.com"'
        }
        baidu{
            buildConfigField 'String','URL','"http://www.baidu.com"'
        }
    }
}
```

需要注意，value这个参数，是单引号中间的部分，尤其对于String类型的值，里面的双引号不能省略。value是什么就写什么，原封不动地放在单引号里。

上面是渠道，productFlavor,其实不光渠道可以配置自定义字段，构建类型BuildType也可以配置。如：

```gradle
android{
    buildTypes{
        debug{
            buildConfigField 'String','NAME','"zhangsan"'
        }
    }
}
```

## 动态添加自定义的资源

实现这一功能的方法是resValue方法。它在BuildType和ProductFlavor这两个对象中存在。它会生成一个资源，效果和在res/values文件中定义一个资源是等价的。

resValue方法有三个参数，第一个是type，也就是你要定义资源的类型，比如有string,id,bool等；第二个是name,也就是定义资源的名称，以便在工程中引用；第三个是value,就是定义资源的值。

```gradle
android{
    productFlavors{
        google{
            resValue 'string','tip','hello'
        }
        baidu{
            resValue 'string','tip','hi'
        }
    }
}
```

## Java编译选项

Android对象提供了一个compileOptions方法，接受一个CompileOptions类型的闭包作为参数，来对Java编译选项进行配置:

```gradle
android{
    compileOptions{
        encoding 'utf-8'
        sourceCompatibility JavaVersion.VERSION_1_6
        targetCompatibility JavaVersion.VERSION_1_6
    }
}
```

CompileOptions是编译配置，它提供三个属性，分别是encoding,sourceCompatibility,targetCompatibility,通过对它们进行设置来配置Java相关的编译选项。

sourceCompatibility是配置Java源代码的编译级别

targetCompatibility是配置生成的Java字节码的版本

## adb操作选项配置

在Android Gradle 中，为我们预留了对adb的一些选项的控制配置，它就是adbOptions{}闭包。

```gradle
android{
    adbOptions{
        timeOutInMs 5*1000
        installOptions '-r','-s'
    }
}
```

## DEX选项配置

Android Gradle 提供了dexOptions{}闭包，让我们可以对dx操作进行一些配置。

```gradle
android{
    dexOptions{
        incremental true
    }
}
```

## 突破65535方法限制

Java源文件被打包成一个DEX文件，这个文件就是优化过的、Dalvik虚拟机可执行的文件，Dalvik虚拟机在执行DEX文件时，使用了short类型来索引DEX文件中的方法，这就意味着单个DEX文件可以被定义的方法最多只有是65535，当定义的方法数量超过时，就会出错。

Android官方给出的解决方案：Multidex。对于Android5.0以后的版本，使用了ART的运行方式，可以天然支持App有多个DEX文件，ART在安装App的时候执行预编译，把多个DEX文件合并成一个oat文件执行。对于Android5.0之前的版本，Dalvik虚拟机限制每个App只能有一个class.dex，要使用它们，就得使用Android提供的Multidex库。

要在项目中使用Multidex,首先要修改Gradle build配置文件，启用Multidex，并同时配置Multidex需要的jar依赖。

```gradle
android{
    defaultConfig{
        multiDexEnabled true
    }
}

dependencies{
    implementation 'com.android.support:multidex:1.0.1'
}
```

配置好之后，开启了Multidex，会让我们的方法多于65535个的时候生成多个DEX文件，名字为classes.dex,classes(...n)这样的形式。但是对于Android5.0以前的系统虚拟机，它只认识一个DEX，名字还是classes.dex,所以想达到程序可以正常运行的目的，也要让虚拟机把其它几个生成的classes加载进来。要做到这步，就必须在App程序启动的入口控制，这个入口就是Application。

Multidex提供现成的Application，名字是MultiDexApplication，如果我们没有自定义Applicaiton的话，直接使用MultiDexApplication即可，在Manifest清单文件中配置：

```xml
<application 
...
android:name="android.support.multidex.MultiDexApplication"
>
...
</application>
```

如果有自定义的Application，并且是直接继承自Application，那么只需要把继承改为我们的MultiDexApplication即可。

如果自定义的Application是继承自第三方提供的Application,就不能改继承了，这个时候可以重写attachBaseContext方法来实现：

```java
@Override
protected void attachBaseContext(Context base){
    super.attachBaseContext(base);
    MultiDex.install(this);
}
```

虽然有了解决65535问题的方法，但还是要尽量避免我们工程中的方法超过65535.首先不能滥用第三方库，如果引用，最好也要自己精简。精简后，还要使用ProGuard减小DEX的大小。还有因为Dalvik linearAlloc的限制，尤其在2.2和2.3版本上，只有5MB，到Android 4.0的时候升级到8MB，所以低于4.0的系统上dexopt的时候可能会崩溃。

## 自动清理未使用的资源

Android Gradle 为我们提供了在构建打包时自动清理未使用的资源的方法，这个就是Resource Shrinking。

Resource Shringking要结合Code Shringking一起使用，即我们开发中经常使用的ProGuard，也就是我们要启用minifyEnabled，是为了减缩代码。

```gradle
android{
    buildTypes{
        release{
            minifyEnabled true
            shringkResources true
            ...
        }
    }
}
```

自动清理未使用的资源这个功能虽然好用，但是有时候会误删有用的程序，因为我们在代码编写的时候，可能会使用反射去引用资源，尤其很多的第三方库会这么做，这个时候Android Gradle就区分不出来，可能会误认为这些资源不有被使用。针对这种情况，Android Gradle提供了keep方法来让我们配置哪些资源不被清理。

keep方法使用非常简单，我们要新建一个xml文件来配置，这个文件是res/raw/keep.xml，然后通过tools:keep属性来设置。这个tools:keep接受一个以逗号分隔的配置资源列表，并且支持星号*通配符。

```xml
<?xml version="1.0" encoding="utf-8">
<resources xmlns:tools="http://schemas.android.com/tools" tools:keep="@layout/layout_a,@layout/layout_b" />
```

keep.xml还有一个属性是tools:shrinkMode，用于配置自动清理资源的模式，默认是false，是安全的。

除了shrinkResources之外，Android Gradle 还提供了一个resConfigs,它属于ProductFlavor的一个方法，可以让我们配置哪些类型的资源才会被打包进APK中。

```gradle
android{
    defaultConfig{
        resConfigs 'zh'
    }
}
```

上面代码表示，只保留zh资源。