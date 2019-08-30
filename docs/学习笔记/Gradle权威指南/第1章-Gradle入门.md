# Gradle入门

## 配置Gradle环境

安装之前确保已经安装配置好Java环境，要求JDK6以上，并且在环境变量里配置了JAVA_HOME，查看Java版本可以在终端输入如下命令：

```java
java -version
```

显示结果如下：

```java
java version "1.8.0_221"
Java(TM) SE Runtime Environment (build 1.8.0_221-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.221-b11, mixed mode)

```

### Linux下搭建Gradle构建环境

先到Gradle官网https://gradle.org下载好Gradle SDK。建议下载all版本，包含了Gradle SDK所有相关内容，包括源代码，文档，示例等。下载后解压。

要运行Gradle，必须把Gradle_HOME/bin目录添加到环境变量PATH中。在Linux下，如果只想为当前登录的用户配置可以运行Gradle，那么可以编辑~/.bashrc文件，添加以下内容:

```java
#这里是我的Gradle目录，换成你自己的
GRADLE_HOME=/home/wangyz/gradle
PATH=${PATH}:${GRADLE_HOME}/bin
Export GRADLE_HOME PATH
```

上面GRADLE_HOME是我的Gradle解压后的目录，这里需要换成你自己的。添加保存后，在终端输入source ~/.bashrc，回车执行让刚才的配置生效。

如果想让所有用户都可以使用Gradle，那么你需要在/etc/profile中添加以上内容，并执行source /etc/profile使配置生效。

现在已经配置好了，要验证我们的配置是否正确，是否可以运行Gradle，只需要打开终端，输入gradle -v命令查看即可，如果能正确显示Gradle版本号，Groovy版本号，JVM等相关信息，那么说明已经配置成功。

```java
------------------------------------------------------------
Gradle 5.6
------------------------------------------------------------

Build time:   2019-08-14 21:05:25 UTC
Revision:     f0b9d60906c7b8c42cd6c61a39ae7b74767bb012

Kotlin:       1.3.41
Groovy:       2.5.4
Ant:          Apache Ant(TM) version 1.9.14 compiled on March 12 2019
JVM:          1.8.0_221 (Oracle Corporation 25.221-b11)
OS:           Linux 4.18.0-15-generic amd64

```

### Windows下搭建Gradle构建环境

通过右击我的电脑，打开属性面板，然后找到环境变量配置项，添加GRADLE_HOME环境变量，然后把GRADLE_HOME/bin添加到PATH系统变量里保存即可。完成后打开CMD，运行gradle -v来进行验证。

## Gradle版Hello World

新建一个目录，我这里是Gradle,然后在该目录下创建一个名为build.gradle的文件。打开编辑该文件，输入以下内容:

```gradle
task hello{
	doLast{
		println 'hello,world!'
	}
}
```

打开终端，然后移动到gradle下，使用gradle -q hello命令来执行构建脚本：

```gradle
gradle -q hello
hello,world!
```

build.gradle是Gradle默认的构建脚本文件，执行Gradle命令的时候，会默认加载当前目录下的build.gradle脚本文件。也可以通过-b参数来指定想要加载执行的文件。

gradle -q hello,这段命令，意思是想执行build.gradle脚本中定义的名为hello的Task，-q参数用于控制gradle输出的日志级别，以及哪些日志可以输出被看到。

在Gradle中，单引号和双引号所包含的内容都是字符串。

## Gradle Wrapper

Wrapper就是对Gradle的一层包装，便于在团队开发过程中统一Gradle构建的脚本，避免因为Gradle版本不统一带来的不必要问题。

### 生成Wrapper

Gradle提供了内置的Wrapper task帮助我们自动生成Wrapper所需的目录文件，在一个项目的根目录下输入gradle wrapper即可生成。

gradlew和gradle.bat分别是Linux和Windows下的可执行脚本，它们的用法和Gradle原生命令是一样的。

### Wrapper配置

当我们在终端执行gradle wrapper生成相关文件的时候，可以为其指定一些参数，来控制Wrapper的生成。

--gradle-version:用于指定使用的Gradle版本

--gradle-distribution-url:用于指定下载Gradle发行版的url地址

如果在调用gradle wrapper的时候，不添加任何参数，那么就会使用当前Gradle版本作为生成的Wrapper的gradle version。

### gralde-wrapper.properties

该配置文件是gradle wrapper的相关配置文件，我们上面执行该任务的任何配置都会被写进该文件。

```gradle
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-5.6-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

### 自定义Wrapper Task

在build.gradle构建文件中加入以下脚本：

```gradle
task mywrapper(type:Wrapper){
	gradleVersion='5.6'
}
```

在这里指定了gradle版本。

## Gradle日志

### 日志级别

Gradle的日志级别和我们使用的大部分语言差不多。除了这些通用的之外，Gradle又增加了QUIET和LIFECYCLE两个级别，用于标记重要以及进度级别的日志信息。

级别 | 用于
-|-
ERROR|错误消息
QUIET|重要消息
WARNING|警告消息
LIFECYCLE|进度消息
INFO|信息消息
DEBUG|调试消息

具体用法

```gradle
#输出QUIET级别及以上的日志信息
gradle -q task
```

### 输出错误堆栈信息

默认情况下，堆栈信息的输出是关闭的。

命令行选项 | 用于
-|-
无选项|没有堆栈信息输出
-s或者-stacktrace|输出关键性的堆栈信息
-S或者--full-stacktrace|输出全部堆栈信息

一般推荐使用-s而不是-S，因为-S输出的堆栈太多太长。不好看。而-s比较精简，可以定位解决我们大部分的问题。

### 自己使用日志信息调试

通常情况下我们一般都是使用print系统方法，把日志信息输出到标准的控制台输出流。

除了print系统方法之外，也可以使用内置的logger更灵活地控制输出不同级别的日志。

```gradle
logger.quiet('quiet日志信息')
logger.error('error日志信息')
logger.warn('warn日志信息')
```

## Gradle命令行

### 使用帮助

查看帮助的方式很简单，基本都是在命令后跟-h或者--help。有的时候会有-?.如：

```gradle
./gradlew -?
./gradle -h
./gradle -help
```

### 查看所有可执行的Tasks

通过运行./gradlew tasks命令。

### Gradle Help任务

Gradle还内置了一个help task,这个help可以让我们了解每一个task的使用帮助，用法是./gradlew help --task.

### 强制刷新依赖

强制刷新很简单，只要在命令行运行的时候加上--refresh-dependencies参数就可以。

### 多任务调用

通过命令行执行多个任务非常简单，只需要按顺序以空格分开就可以了。如./gradlew clean jar。

### 通过任务名字缩写执行

Gradle提供了基于驼峰命名法的缩写调用。比如connectCheck，执行的时候可以使用./gradlew connectTask,也可以使用./gradlew cC的方法来执行。

