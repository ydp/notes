---
title: scala入门
date: 2016-09-08 20:24:21
categories: 语言
tags: [scala, 入门]
---

mac上安装
``` bash
brew install scala
```

配置vim对scala高亮，使用Plugin的方式
``` bash
Plugin 'derekwyatt/vim-scala'
:PluginInstall
. ~/.vimrc
```

编写helloworld, 新建文件 hello.scala
``` scala
object HelloWorld {
    def main(args: Array[String]): Unit = {
        println("Hello, World!")
    }
}
```
运行：
``` scala
scala hello.scala
```

#### 编译
``` bash
scalac hello.scala
scala HelloWorld
```

还可以给scala指定选项， -d 指定生成的class文件目录

``` bash
scalac -d classes hello.scala
```

也可以指定类路径，比如刚刚的类生成在classes文件夹下，运行就这么做：
``` bash
scala -cp classes HelloWorld
```

注意： scala的参数必须是top-level对象，如果继承了trait App，那么对象里的所有语句都会被执行，如下。
``` scala
object HelloWorld extends App {
	println("Hello, World!")
}
```

否则的话，就要像前面的例子一样定义main方法。

#### 交互式运行
在shell中输入代码：

``` bash
localhost:~ yuandingping$ scala
Welcome to Scala 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_91).
Type in expressions for evaluation. Or try :help.

scala> object HelloWorld {
     |     def main(args: Array[String]): Unit = {
     |         println("Hello, World!")
     |     }
     | }
defined object HelloWorld

scala> HelloWorld.main(Array())
Hello, World!

scala> q
<console>:12: error: not found: value q
       q
       ^

scala> :q
localhost:~ yuandingping$
```

:q 表示退出scala环境。


#### 脚本运行

scala还可以运行在脚本中， 比如script.sh:

``` bash
#!/usr/bin/env scala
object HelloWorld extends App {
	println("Hello, World!")
}
```
运行：
``` bash
./script.sh
```
注意： script.sh必须有可执行权限，且scala命令加入了类路径

貌似下面的执行方法是不对的:

``` bash
sh script.sh
```
但是可以这样:

``` bash
scala script.sh
```
以上
[原文链接](http://www.scala-lang.org/documentation/getting-started.html)


scala一般使用sbt管理工程，执行编译构建等工作:
``` bash
brew install sbt
```
这里有很多scala的学习资料： 

* [scala课堂](https://twitter.github.io/scala_school/zh_cn/)
* [官方文档](http://docs.scala-lang.org/zh-cn/overviews/) 
* [A Tour of Scala](http://docs.scala-lang.org/tutorials/)







