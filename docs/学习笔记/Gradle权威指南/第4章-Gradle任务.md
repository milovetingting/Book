# Gradle任务

## 多种方式创建任务

1、直接以一个任务名字创建一个任务的方式:

```gradle
def Task task1 = task(task1)
task1.doLast{
    println 'task1'
}
```

这种方式的创建其实是调用Project对象中的task(String name)方法。该方法的完整定义:

```gradle
Task task(String name) throws InvalidUserDataException
```

2、以一个任务名字+一个对该任务配置的Map对象来创建任务:

```gradle
def Task task2 = task(task2,group:BasePlugin.BUILD_GROUP)

task2.doLast{
    println 'task2'
}
```

Task参数Map可用配置

配置项 | 描述 | 默认值
-|-|-
type | 基于一个存在的Task来创建，和我们类继承差不多 | DefaultTask
overwrite | 是否替换存在的Task，这个和type配合起来用 | false
dependsOn | 用于配置任务的依赖 | []
action | 添加到任务中的一个Action或者闭包 | null
description | 用于配置任务的描述 | null
group | 用于配置任务的分组 | null

3、任务名字+闭包配置的方式：

```gradle
task task3{
    description 'task3'
    doLast{
        println 'task3'
        println "任务描述:${description}"
    }
}
```

Map配置的项有限，所以可以通过闭包的方式进行更加灵活的配置。闭包里的委托对象就是Task，所以你可以使用Task对象的任何方法，属性等信息。

TaskContainer创建任务的方式：

```gradle
tasks.create("task4"){
    description 'task4'
    doLast{
        println 'task4'
        println "任务描述:${description}"
    }
}
```

tasks是Project对象的属性，其类型是TaskContainer，可以用它来直接创建任务。

## 多种方式访问任务

创建的任务都会作为项目的一个属性，属性名就是任务名，所以可以直接通过任务名称来访问和操作任务:

```gradle
task task5

task5.doLast{
    println 'task5:doLast'
}
```

任务都是通过TaskContainer创建的，其实TaskContainer就是我们创建的集合。在Project中可以通过tasks属性访问TaskContainer，所以可以通过访问集合的方式来访问创建的任务：

```gradle
task task6

tasks['task6'].doLast{
    println 'task6:doLast'
}
```

通过路径来访问。访问方式有两种，一种是get,一种是find，区别在于get如果找不到任务会抛出UnKnownTaskException异常，而find在找不到任务时返回null。

```gradle
task task7

tasks['task7'].doLast{
    println tasks.findByPath(':Chapter4:task7')
    println tasks.getByPath(':Chapter4:task7')
    println tasks.findByPath('abc')
}
```

通过名称访问。方式也有两种：get和find，区别和路径方式相同:

```gradle
task task8

tasks['task8'].doLast{
    println tasks.findByName('task8')
    println tasks.findByName('task8')
    println tasks.findByName('abc')
}
```

通过路径访问的时候，参数值可以是任务路径，也可以是任务名字。而通过名称访问，参数只能是任务名称,不能是路径。

## 任务分组和描述

任务是可以分组和添加描述的。任务分组其实就是对任务分类，便于对任务归类整理。任务的描述就是说明任务有什么用，是任务的大概说明。

```gradle
task task9{
    group BasePlugin.BUILD_GROUP
    description '构建任务'
    doLast{
        println 'task9:doLast'
    }
}
```

## <<操作符

在Gradle 5.1后已经废弃。

## 任务的执行分析

当我们执行一个任务的时候，其实就是执行其拥有的actions列表。这个列表保存在Task的对象实例中的actions成员变量中，其类型是List。

## 任务排序

通过任务的shouldRunAfter和mustRunAfter这两个方法，可以控制一个任务应该或者一定要在某个任务之后执行。

```gradle
task task12{
    doLast{
        println 'task12'
    }
}

task task13{
    doLast{
        println 'task13'
    }
}

task12.mustRunAfter task13
```

## 任务的启用和禁用

Task中有个enabled属性，用于启用和禁用任务，默认为true,表示启用，设置为false，则禁止任务执行，输出会提示该任务被跳过。

```gradle
task task14 {
    doLast{
        println 'task14'
    }
}

task14.enabled = false
```

## 任务的onlyIf断言

Task有一个onlyIf方法，它接受一个闭包作为参数，如果该闭包返回true,则该任务执行，否则跳过。

以打渠道包为例。首发应用宝和百度，直接编译会打出所有包，执行时间长，不符合需求，可以采用onlyIf来控制：

```gradle
final String BUILD_APP = "build_app"
final String BUILD_APPS_ALL = "all"
final String BUILD_APPS_SHOUFA = "shoufa"
final String BUILD_APPS_EXCLUDE_SHOUFA = "exclude_shoufa"

task(QQRelease).doLast{
    println '打应用宝的包'
}

task(BaiduRelease).doLast{
    println '打百度的包'
}

task(HuaWeiRelease).doLast{
    println '打华为的包'
}

task(MIUIRelease).doLast{
    println '打MIUI的包'
}

task build{
    group BasePlugin.BUILD_GROUP
    description "打渠道包"
}

build.dependsOn QQRelease,BaiduRelease,HuaWeiRelease,MIUIRelease

QQRelease.onlyIf{
    def execute = false
    if(project.hasProperty(BUILD_APP))
    {
        Object buildApp = project.property(BUILD_APP)
        if(BUILD_APPS_SHOUFA.equals(buildApp)||BUILD_APPS_ALL.equals(buildApp))
        {
            execute = true
        }
        else{
            execute = false
        }
    }
    else{
        execute = true
    }
    execute
}

BaiduRelease.onlyIf{
    def execute = false
    if(project.hasProperty(BUILD_APP))
    {
        Object buildApp = project.property(BUILD_APP)
        if(BUILD_APPS_SHOUFA.equals(buildApp)||BUILD_APPS_ALL.equals(buildApp))
        {
            execute = true
        }
        else{
            execute = false
        }
    }
    else{
        execute = true
    }
    execute
}

HuaWeiRelease.onlyIf{
    def execute = false
    if(project.hasProperty(BUILD_APP))
    {
        Object buildApp = project.property(BUILD_APP)
        if(BUILD_APPS_EXCLUDE_SHOUFA.equals(buildApp)||BUILD_APPS_ALL.equals(buildApp))
        {
            execute = true
        }
        else{
            execute = false
        }
    }
    else{
        execute = true
    }
    execute
}

MIUIRelease.onlyIf{
    def execute = false
    if(project.hasProperty(BUILD_APP))
    {
        Object buildApp = project.property(BUILD_APP)
        if(BUILD_APPS_EXCLUDE_SHOUFA.equals(buildApp)||BUILD_APPS_ALL.equals(buildApp))
        {
            execute = true
        }
        else{
            execute = false
        }
    }
    else{
        execute = true
    }
    execute
}
```

执行方式如下:

```gradle
#打所有渠道包
gradle build
gradle -Pbuild_app=all build
#打首发包
gradle -Pbuild_app=shoufa build
#打非首发包
gradle -Pbuild_app=exclude_shoufa build
```

命令行中-P意思是为Project指定K-V格式的属性键值对，格式为-PK=V。

## 任务规则

```gradle
tasks.addRule("对规则的描述"){
    String taskName->
        task(taskName) {
            println "${taskName}任务不存在"
        }
}

task task15{
    dependsOn missTask
}
```