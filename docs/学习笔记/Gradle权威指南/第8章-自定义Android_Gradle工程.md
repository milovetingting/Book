# 自定义Android Gradle工程

## defaultConfig默认配置

defaultConfig是Android对象中的一个配置项，负责定义所有的默认配置。一个基本的defaultConfig配置如下：

```gradle
android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    defaultConfig{
        applicationId "com.wangyz.app"
        minSdkVersion 14
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
        //...
    }
}
```

### applicationId

applicationId是ProductFlavor的一个属性，用于指定生成的App的包名，默认情况下是Null.这个时候在构建的时候，会从我们的AndroidManifest.xml文件读取，也就是我们在AndroidManifest.xml文件中配置的manifest标签的package属性值。

### minSdkVersion

minSdkVersion是ProductFlavor的一个方法，对应的方法原型为：

```gradle
public void minSdkVersion(int minSdkVersion){
    setMinSdkVersion(minSdkVersion);
}
```

它可以指定我们的App最低支持的Android操作系统版本，其对应的值是Android Sdk的API LEVEL.它还有两个方法原型:

```gradle
public void setMinSdkVersion(@Nullable String minSdkVersion){
    setMinSdkVersion(getApiVersion(minSdkVersion))
}

public void minSdkVersion(@Nullable String minSdkVersion){
    setMinSdkVersion(minSdkVersion)
}
```

### targetSdkVersion

这个用于配置我们基于哪个Android SDK开发，它的可选值和minSdkVersion一样。没有配置的时候，也会从AndroidManifest.xml读取。

### versionCode

它也是ProductFlavor的一个属性，用于配置Android App的内部版本号，是一个整数，通常用于版本的升级，没有配置的时候，从AndroidManifest.xml读取。方法原型是:

```gradle
@NonNull
public ProductFlavor setVersionCode(Integer versionCode){
    mVersionCode = versionCode;
    return this;
}

@Override
@NonNull
public Integer getVersionCode(){
    return mVersionCode;
}
```

### versionName

用于配置Android App的版本名称，如V1.0.0等。

### testApplicationId

用于配置测试App的包名，默认情况下是applicationId+".test"。

### testInstrumentationRunner

用于配置单元测试时使用的Runner，默认使用的是android.test.InstrumentationTestRunner。

### signingConfig

配置默认的签名信息，对生成的App签名。

### proguardFile

用于配置App ProGuard混淆所使用的Proguard配置文件。

### proguardFiles

这个也是配置ProGuard的配置文件，只不过它可以同时接受多个配置文件，因为它的参数是一个可变类型的参数。

## 配置签名信息

```gradle
android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    signingConfigs{
        release{
            storeFile file("myrelease.keystore")
            storePassword "password"
            keyAlias "MyReleaseKey"
            keyPassword "password"
        }
    }
}
```

上面例子中，配置了一个名为release的签名配置，除此之外，还可以配置多个不同的签名信息。

```gradle
android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    signingConfigs{
        release{
            storeFile file("myrelease.keystore")
            storePassword "password"
            keyAlias "MyReleaseKey"
            keyPassword "password"
        }
        debug{
            storeFile file("mydebug.keystore")
            storePassword "password"
            keyAlias "MyDebugKey"
            keyPassword "password"
        }
    }
}
```

现在已经配置好了两个签名信息，但还没有被应用，应用方法如下:

```gradle
android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    signingConfigs{
        release{
            storeFile file("myrelease.keystore")
            storePassword "password"
            keyAlias "MyReleaseKey"
            keyPassword "password"
        }
        debug{
            storeFile file("mydebug.keystore")
            storePassword "password"
            keyAlias "MyDebugKey"
            keyPassword "password"
        }
    }

    defaultConfig{
        applicationId "com.wangyz.app"
        minSdkVersion 14
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
        signingConfig signingConfigs.debug
    }
}
```

除了上面的默认签名配置外，也可以对构建类型分别配置签名信息。


```gradle
android{
    compileSdkVersion 23
    buildToolsVersion "23.0.1"

    signingConfigs{
        release{
            storeFile file("myrelease.keystore")
            storePassword "password"
            keyAlias "MyReleaseKey"
            keyPassword "password"
        }
        debug{
            storeFile file("mydebug.keystore")
            storePassword "password"
            keyAlias "MyDebugKey"
            keyPassword "password"
        }
    }

    buildTypes{
        release{
            signingConfig signingConfigs.release
        }
        debug{
            signingConfig signingConfigs.debug
        }
    }
}
```

## 构建的应用类型

如果想增加新的构建类型，在buildTypes{}代码块中继续添加元素就可以。

### applicationIdSuffix

applicationIdSuffix是BuildType的一个属性，用于配置基于默认的applicationId的后缀。

### debuggable

debuggable用于配置一个可供调试的apk。其值可以true或false。

### jniDebuggable

用于配置是否生成一个可供调试jni代码的apk。可接受boolean类型的值。

### minifyEnabled

用于配置该BuildType是否启用Proguard混淆，接受boolean类型的值。

### multiDexEnabled

用于配置该BuildType是否启用自动拆分多个Dex的功能。

### proguardFile

用于配置Proguard混淆使用的配置文件。

### proguardFiles

用于配置Proguard混淆使用的配置文件，可同时配置多个Proguard配置文件。

### shrinkResources

用于配置是否自动清理未使用的资源，默认为false。

### signingConfig

配置该BuildType使用的签名配置。

每一个BuildType都会生成一个SourceSet，默认位置为src//。新增的BuildType名字不能是main和androidTest,因为这两个已经被系统占用，同时每个BuildType之间名称不能相同。

## 使用混淆

代码混淆是一个非常有用的功能，它不仅能优化代码，让apk包变得更小，还可以混淆原来的代码，让反编译的人不容易看明白业务逻辑。

```gradle
android{
    buildTypes{
        release{
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),'proguard-rules.pro'
        }
        debug{

        }
    }
}
```

## 启用Zipalign优化

zipalign是Android为我们提供的一个整理优化apk文件的工具。它能提高系统和应用的运行效率，更快地读写apk中的资源，降低内存的使。

```gradle
android{
    buildTypes{
        release{
            zipAlignEnabled true
        }
        debug{
            
        }
    }
}
```