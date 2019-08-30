# Groovy基础

Groovy是基于JVM虚拟机的一种动态语言。每个Gradle的build脚本文件都是一个Groovy脚本文件。

## 字符串

在Groovy中，分号不是必需的。在Groovy中，单引号和双引号都可以定义一个字符串变量 ，单引号标记的是纯粹的字符串变量，而不是对字符串里的表达式做运行，但是双引号可以。

```gradle
task printString {
	def str1 = '单引号'
	def str2 = "双引号"
	
	println "单引号定义的字符串类型:"+str1.getClass().name
	println "双引号定义的字符串类型:"+str2.getClass().name
}
```

输出结果:

```gradle
单引号定义的字符串类型:java.lang.String
双引号定义的字符串类型:java.lang.String
```

而双引号可以做运算:

```gradle
task  printStringVar{
	def name = '张三'

	println '单引号的变量计算:${name}'
	println "双引号的变量计算:${name}"
}
```

输出结果:

```gradle
单引号的变量计算:${name}
双引号的变量计算:张三
```

一个$符号紧跟着一对花括号，花括号里放表达式，如${name}、${1+1}等，只有一个变量的时候，可以省略花括号，如$name。

## 集合

### List

```gradle
task printList {
	def numList = [1,2,3,4,5]
	println numList.getClass().name

	println numList[1]//访问第二个元素
	println numList[-1]//访问最后一个元素
	println numList[-2]//访问倒数第二个元素
	println numList[1..3]//访问第二个到第四个元素

	numList.each {
		println it
	}
}
```

Groovy还为List提供了非常方便的迭代操作，这就是each方法。

### Map

Map用法和List想像，只不过它的值是一个K:V键值对。访问也非常灵活，采用map[key]或者map.key都可以。

```gradle
task printMap{
	def map1 = ['name':'张三','age':18]
	println map1.getClass().name

	println map1['name']
	println map1.age

	map1.each{
		println "key:${it.key},Value:${it.value}"
	}
}
```

## 方法

### 括号可以省略

```gradle
task invokeMethod{
	method1(1,2)
	method1 1,2
}

def method1(int a,int b){
	println a+b
}
```

### return可以不写

在Groovy中，定义有返回值的方法时，return语句不是必需的。当没有return时，Groovy会把方法执行过程中的最后一句代码作为返回值。

```gradle
task printMethodReturn{
	def max1 = method2 1,2
	def max2 = method2 3,5
	println "max1:${max1},max2:${max2}"

}

def method2(int a,int b){
	if(a>b){
		a	
	}else{
		b
	}
}
```

### 代码块可以作为参数传递

## JavaBean

```gradle
task helloJavaBean{
	Person p = new Person()
	println "名字是:${p.name}"
	p.name="张三"
	println "名字是:${p.name}"
	println "年龄是:${p.age}"
}

class Person{
	private String name

	public int getAge(){
		18
	}
}
```

在Groovy中，并不是一定要定义成员变量才能作为类的属性访问。我们直接用getter/setter方法，也一样可以当作属性访问。

## 闭包

闭包是Groovy的一个非常重要的特性，是DSL的基础。

### 初识闭包

```gradle
task helloClosure{
	customEach{
		println it
	}

	eachMap{k,v->println "${k} is ${v}"}
}

def customEach(closure){
	for(int i in 1..10){
		closure(i)
	}
}
```

### 向闭包传递参数

当闭包有一个参数时，默认就是it，当有多个参数时，it就不能表达了，我们需要把参数一一列出。

```gradle
def eachMap(closure){
	def map1 = ["name":"张三","age":18]
	map1.each{
		closure(it.key,it.value)
	}
}
```

### 闭包委托

Groovy的闭包有thisObject,owener,delegate三个属性。默认情况下，delegate和owner是相等的，但是delegate是可以被修改的。

thisObject的优先级最高，thisObject其实就是这个构建脚本的上下文，它和脚本中的this对象是相等的。优先级从高到低依次是：thisObject>owner>delegate。

在DSL中，比如Gradle，我们一般会指定delegate为当前的it，这样我们在闭包内就可以对该it进行配置，如下:

```gradle
task configClosure{
	person{
		name="张三"
		age = 18
		dumpPerson()
	}
}

class Person{
	String name
	int age

	def dumpPerson(){
		println "name:${name},age:${age}"
	}
}

def person(Closure<Person> closure){
	Person p = new Person()
	closure.delegate = p
	closure.setResolveStrategy(Closure.DELEGATE_FIRST)
	closure(p)
}
```

## DSL

DSL,即Domain Specific Language，领域特定语言，就是专门关注某一领域的语言，在于专，而不是全。

Gradle就是一门DSL,它是基于Groovy，专门解决自动化构建的DSL。