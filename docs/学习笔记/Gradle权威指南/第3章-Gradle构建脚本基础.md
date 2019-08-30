# Gradle构建脚本基础

## Settings文件

在Gradle中，定义了一个设置文件，用于初始化以及工程树的配置。设置文件的默认名为settings.gradle,放在根工程目录下。

设置文件大多数的作用都是为了配置子工程。根工程相当于Android Studio中的Project，一个根工程可以有很多子工程。

一个子工程只有在Settings文件里配置了Gradle才会识别，才会在构建的时候被包含进去。

```gradle
rootProject.name = 'android-gradle'

include ':Chapter1'
project(':Chapter1').projectDir = new File(rootDir,'Chapter1')

include ':Chapter2'
project(':Chapter2').projectDir = new File(rootDir,'Chapter2')

include ':Chapter3'
project(':Chapter3').projectDir = new File(rootDir,'Chapter3')
```

上面的配置，定义了一些子项目，并且为它们指定了相应的目录。如果不指定，则默认为同级目录。利用这个特性，我们可以把我们的工程放到任何目录下，可以非常灵活地对我们的工程进行分级，分类等。只要在Settings文件里指定好路径就可以了。

## Build文件

每个Project都会有一个Build文件，该文件是该Project构建的入口，可以在这里对Project进行配置，比如配置版本，需要哪些插件，依赖哪些库等。

Root Project也有自己的Build文件。Root Project 可以取到所有的Child Project，所以在Root Project的Build文件里可以对Child Project统一配置，如应用的插件，依赖的Maven中心库等。比如配置所有的Child Project的仓库为jcenter：

```gradle
subprojects{
    repositories{
        jcenter()
    }
}
```

又比如，开发一个大型的Java工程，该工程被分为很多小模块，每个模块都是一个Child Project，这些模块也是Java工程，这种情况下可以统一配置：

```gradle
subprojects{
    apply plugin:"java"
    repositories{
        jcenter()
    }
}
```

除了subprojects外，还有allprojects,用于对所有工程进行配置。

## Projects以及Tasks

一个Project包含很多个Task，也就是说每个Project是由多个Task组成的。

## 创建一个任务

```gradle
task customTask1{
	doFirst{
		println 'customTask1:doFirst'
	}
	doLast{
		println 'customTaks1:doLast'
	}
}
```

创建任务的另一种方式:

```gradle
tasks.create("customTask2"){
	doFirst{
		println 'customTask2:doFirst'
	}
	doLast{
		println 'customTask2:doLast'
	}
}
```

## 任务依赖

任务之间是可以有依赖关系的，这样我们就能控制哪些任务优先于哪些任务执行。

创建任务的时候，通过dependsOn可以指定依赖的任务

```gradle
task customTask3(dependsOn:customTask2){
	doLast{
		println 'customTask3:doLast'
	}
}
```

另外，一个任务可以依赖多个任务

```gradle
task customTask4{
	dependsOn customTask3,customTask1
	doLast{
		println 'customTask4:doLast'
	}
}
```

## 任务间通过API控制、交互

要使用任务名操作任务，必须先定义声明，因为脚本是顺序执行的。

```gradle
task customTask5{
	println 'customTask5'
}

customTask5.doFirst{
	println 'customTask5:doFirst'
}

customTask5.doLast{
	println 'customTask5:doLast'
}
```

## 自定义属性

Project和Task都允许用户添加额外的自定义属性，要添加自定义属性，通过应用所属对应的ext即可实现。添加之后，通过ext属性可以读取和设置，如果要同时添加多个自定义属性，可以通过ext代码块来实现。

```gradle
ext.name='张三'

ext{
	age = 18
	address = '中国'
}

task customTask6{
	println "姓名是:${name},年龄是:${age},地址是：${address}"
}
```

相比局部变量，自定义属性有更为广泛的作用范围。可以跨Project，跨Task访问这些自定义的属性。只要能访问到这些属性所属的对象，这些属性就可以被访问到。

自定义属性不仅仅局限在Project和Task上，还可以应用在SourceSet中。

```gradle
apply plugin:"java"

sourceSets.all{
	ext.resourcesDir = null
}

sourceSets{
	main{
		resourcesDir= 'main/res'
	}
	test{
		resourcesDir = 'test/res'
	}
}

task customTask7{
	sourceSets.each{
		println "${it.name}的resourcesDir是:${it.resourcesDir}"
	}
}
```

在项目中，一般使用它来自定义版本和版本名称，把版本号和版本名称单独放在一个Gradle文件中。

## 脚本即代码，代码即脚本
