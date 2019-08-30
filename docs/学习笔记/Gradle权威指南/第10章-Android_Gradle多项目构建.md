# Android Gradle 多项目构建

## Android 项目区别

Android项目一般分为库项目，应用项目，测试项目，Android Gradle 根据这些项目分别对应3种插件：com.android.library,com.android.application,com.android.test。

## Android多项目设置

定义一个工程，包含很多项目，在Gradle中，项目的结构没有那么多限制，只要在settings.gradle里配置好这些项目就可以了。

## 库项目引用的配置

Android库项目的引用，通过dependencies实现:

```gradle
dependencies{
    implements project(':plugin')
}
```

## 库项目单独发布

### Maven私服搭建

搭建自己的Maven私服，推荐使用Nexus Repositories Manager。

具体的搭建如下：

1、下载。在https://www.sonatype.com/ 选择对应的软件类型，我这里选择的是OSS3版本，即免费版。 在https://www.sonatype.com/download-nexus-repo-oss 页面根据操作系统选择需要下载的应用。

2、解压。解压后有两个文件夹，nexus-3.13.0-01和sonatype-work。

3、启动。进入nexus-3.13.0-01目录下的bin目录，然后在命令行中输入./nexus start，启动nexus。

4、浏览器访问http://localhost:8081 ,如访问成功，即表示nexus搭建成功。以默认的管理员帐号admin,密码admin123登录，可以看到默认创建的仓库。

nexus的具体配置这里不展开讲，具体可以在网上找相关资源，这里只用默认配置。

### 库项目发布

新建名为TestLib的Android Library，在根目录的gradle.properties中配置如下(这里配置是为了方便统一管理，也可以直接写在library的build.gradle中):

```gradle
# maven local config
#正式版本号
versionName=1.0.0
#快照版本号
snapshotVersionName=1.0-SNAPSHOT
#快照仓库地址
mavenSnapshotUrl=http://localhost:8081/repository/maven-snapshots/
#发布仓库地址
mavenReleasesUrl=http://localhost:8081/repository/maven-releases/
maven_local_username=admin
maven_local_password=admin123
#项目组 id
maven_pom_groupId=com.wangyz.plugins
#项目名称
maven_pom__artifactId=testlib
#打包类型
maven_pom__packaging=aar
maven_pom__description=test upload
```

在TestLib目录下的build.gradle的android节点下增加以下配置：

```gradle
// type显示指定任务类型或任务, 这里指定要执行Javadoc这个task,这个task在gradle中已经定义
    task androidJavadocs(type: Javadoc) {
        // 设置源码所在的位置
        source = android.sourceSets.main.java.sourceFiles
    }

    // 生成javadoc.jar
    task androidJavadocsJar(type: Jar) {
        // 指定文档名称
        classifier = 'javadoc'
        from androidJavadocs.destinationDir
    }

    // 生成sources.jar
    task androidSourcesJar(type: Jar) {
        classifier = 'sources'
        from android.sourceSets.main.java.sourceFiles
    }

    // 产生相关配置文件的任务
    artifacts {
        archives androidSourcesJar
        archives androidJavadocsJar
    }

    //上传 到 maven 的任务
    uploadArchives {
        repositories.mavenDeployer {

            repository(url: mavenReleasesUrl) {
                authentication(userName: maven_local_username, password: maven_local_password)
            }

            snapshotRepository(url: mavenSnapshotUrl) {
                authentication(userName: maven_local_username, password: maven_local_password)
            }

            pom.project {
                // 注意：【这里通过切换 versionName 的赋值来区分上传快照包还是正式包（snapshot 版本必须以 -SNAPSHOT 结尾）】
                //version snapshotVersionName
                version versionName
                artifactId maven_pom__artifactId
                groupId maven_pom_groupId
                packaging maven_pom__packaging
                description maven_pom__description
            }
        }

    }
```

命令行切换到TestLib目录下，执行gradle uploadArchives命令，执行成功后，在浏览器中可看到上传成功。

### 库项目的引用

在要引用的项目，如app，在项目根目录的build.gradle的allprojects节点中添加以下配置:

```gradle
allprojects {
    repositories {
        google()
        jcenter()

        mavenCentral()
        mavenLocal()

        maven {
            url mavenReleasesUrl
        }

        maven {
            url mavenSnapshotUrl
        }

        maven {
            url 'https://maven.google.com'
        }
        
    }
}
```

然后在app的build.gradle中引入依赖:

```gradle
dependencies {
    implementation 'com.wangyz.plugins:testlib:1.0'
}
```

同步项目后，即可引用TestLib的相关资源。

参考以下资源，在此表示感谢！

https://www.jianshu.com/p/33d9861217bf