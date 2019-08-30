# Gradle插件

## 插件的作用

把插件应用到项目中，插件会扩展项目的功能，帮助在项目构建过程中做很多事情。

1、可以添加任务到项目中，帮助完成测试、编译、打包等。

2、可以添加依赖配置到项目中，可以通过它们配置项目在构建过程中需要的依赖，如编译时依赖的第三方库等。

3、可以向项目中现有的对象类型添加新的扩展属性、方法等。

4、可以对项目进行一些约定，如应用Java插件后，约定src/main/java目录是我们的源代码存在位置，在编译的时候也是编译这个目录下的Java源代码文件。

## 如何应用一个插件

插件的应用都是通过Project.apply()方法完成的。

### 应用二进制插件

二进制插件就是实现了org.gradle.api.Plugin接口的插件，它们可以有Plugin id。

```gradle
apply plugin:'java'
```

上面的语句，其中'java'就是Java插件的plugin id,它是唯一的。其实它对应的类型是org.gradle.api.plugins.JavaPlugin,所以通过该类型，我们也可以应用这个插件：

```gradle
apply plugin:org.gradle.api.plugins.JavaPlugin
```

又因为包org.gradle.api.plugins是默认导入的，所以可以去掉包名直接写成:

```gradle
apply plugin:JavaPlugin
```

### 应用脚本插件

build.gradle

```gradle
apply from:'version.gradle'
task task1{
    println "版本是:${versionName},版本号是:${versionCode}"
}
```

version.gradle

```gradle
ext{
    versionName = '1.0'
    versionCode = 1
}
```

### apply方法的其他用法

Project.apply()方法有三种使用方法：

```gradle
void apply(Map<String,?> options);
void apply(Closure closure);
void apply(Action<? super ObjectConfigurationAction> action);
```

闭包的方式如下：

```gradle
apply{
    plugin 'java'
}
```

Action方式:

```gradle
apply (new Action<ObjectConfigurationAction>){
    @Override
    void execute(ObjectConfigurationAction objectConfigurationAction){
        objectConfigurationAction.plugin('java')
    }
}
```

### 应用第三方发布的插件

第三方发布的jar的二进制插件，我们在应用的时候，必须要先在buildscript{}里配置其classpath才能使用。

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

buildscript{}是一个构建项目前，为项目进行前期准备和初始化相关配置依赖的地方，配置好所需的依赖，就可以应用插件了

```gradle
apply plugin:'com.android.application'
```


### 使用plugins DSL应用插件

```gradle
plugins{
    id 'java'
}
```

### 更多好用的插件

可以在https://plugins.gradle.org/ 上找到，也可以在github上找。

## 自定义插件

自定义插件必须实现Plugin接口，这个接口只有一个apply方法，该方法在插件被应用的时候执行。

定义在build脚本文件里：

```gradle
apply plugin:CustomPlugin

class CustomPlugin implements Plugin<Project>{
    void apply(Project project){
        project.task('customTask').doLast{
            println '这是一个通过自定义插件方式创建的任务。'
        }
    }
}
```

这种只能在自己项目里用，如果想开发一个独立的插件给所有想用的人，则需要创建单独工程来开发自定义插件了。

新建一个Android Module

清空Module的build.gradle内容，添加以下内容,配置开发所需的依赖:

```gradle
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}
```

然后实现依赖类:

删除src/main目录下的所有文件，新建一个groovy文件夹,在这个文件夹新建包，如com.wangyz.plugins,然后在这个文件夹下，新建一个类，如:CustomPlugin.groovy,内容如下:

```groovy
package com.wangyz.plugins

import org.gradle.api.Plugin
import org.gradle.api.Project

class CustomPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        project.task('CustomTask').doLast {
            println("这是一个通过自定义插件方式创建的任务")
        }
    }
}
```

在src/main文件夹下新建resources文件夹,然后在这个文件夹中新建META-INF文件夹，然后在这个文件夹下新建gradle-plugins文件夹,然后新建com.wangyz.plugins.customplugin.properties文件，文件名就是其它应用依赖的名。内容如下:

```gradle
implementation-class=com.wangyz.plugins.CustomPlugin
```

写好后，我们配置发布：

在Module的build.gradle文件中，添加以下内容:

```gradle
apply plugin: 'maven-publish'

publishing {
    publications {
        mavenJava(MavenPublication) {

            groupId 'com.wangyz.plugins'
            artifactId 'customplugin'
            version '1.0.0'

            from components.java

        }
    }
}

publishing {
    repositories {
        maven {
            // change to point to your repo, e.g. http://my.org/repo
            url uri('/home/wangyz/repos')
        }
    }
}
```

然后在控制台，输入以下指令:

```gradle
./gradlew publish
```

发布成功后，配置引用：

在需要引入依赖的工程根目录下的build.gradle添加以下内容:

```gradle
buildscript {
    repositories {
        maven {
            //local maven repo path
            url uri('/home/wangyz/repos')
        }
        
    }
    dependencies {
        //这里配置为发布时填写的:groupId:artifactId:version
        classpath 'com.wangyz.plugins:customplugin:1.0.0'
    }
}
```

在App的build.gradle下添加以下内容：

```gradle为properties的文件名
//这里的配置为:
apply plugin: 'com.wangyz.plugins.customplugin'
```