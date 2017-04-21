# Spark环境搭建
干这个之前默认的你的机器上已经装好了hadoop，顺带说一句Spark的Python API现在还不支持3.6，所以。。。不过对于之前新建的那个hadoop用户的话，这个用户默认的python应该还是ubuntu(如果是ubuntu16.04的话)自带的2.7.3和3.5.2，所以就无所谓啦，不会用到主账户的anaconda里面的3.6或者其他的什么乱七八糟。

如果以后有用Python写程序在Spark上跑，会用到pandas之类的包的话。。再研究怎么弄吧。

**注意**该文档下所有指令都默认在hadoop用户下执行，也就是说你需要先

`$ su hadoop`或者`$ su [某个你自己设置的名字]`
## Step1: 验证Java是否安装
`$ java -version`

如果安装了，大概会显示这个:
```
openjdk version "1.8.0_121"
OpenJDK Runtime Environment (build 1.8.0_121-8u121-b13-0ubuntu1.16.04.2-b13)
OpenJDK 64-Bit Server VM (build 25.121-b13, mixed mode)
```
没装的话回顾hadoop环境搭建。

## Step2: 验证Scala是否安装
`$ scala -version`

如果安装了大概会显示这个

`Scala code runner version 2.11.6 -- Copyright 2002-2013, LAMP/EPFL`

顺便提一下我的电脑上显示的是这个
```
cat: /usr/lib/jvm/java-8-openjdk-amd64/release: No such file or directory
Scala code runner version 2.12.1 -- Copyright 2002-2016, LAMP/EPFL and Lightbend, Inc.
```
很蒙，不知道为什么会有这个找不到路径的错误(我的jdk下面确实是没有一个叫release的文件夹)，不过Spark运行正常，如果有同样的错误出现的话，暂时先不理它好了。
## Step3：下载Scala
好吧我知道你没装Scala，那就点这个[链接](http://www.scala-lang.org/download/)下一个吧。
或者

`$ wget https://downloads.lightbend.com/scala/2.12.1/scala-2.12.1.tgz`

这个资源下载速度是真的慢，如果下不动就百度一下换个地方下载好了。然后切换到下载的目录,然后
```
$ tar zxvf scala-2.12.1
$ sudo mv scala-2.12.1 /usr/local/scala
```
在bashrc里面添加这个:
```
$ vim ~/.bashrc
export PATH=$PATH:/usr/local/scala/bin
```
然后再

`$ source ~/.bashrc `

或者直接这样也行：

`$ export PATH=$PATH:/usr/local/scala/bin`

## Step 5:下载并安装Spark
进[这里](http://spark.apache.org/downloads.html)下吧，然后解压：

`$ tar zxvf spark-2.1.0-bin-hadoop2.7.tgz`

移动spark文件夹

`$ mv spark-2.1.0-bin-hadoop2.7 /usr/local/spark`

设置环境变量：

`$ vim ~/.bashrc`

添加一下PATH：

`export PATH=$PATH:/usr/local/spark/bin`

最后再

`$ source ~/.bashrc`

## Step 6: 验证Spark

`$ spark-shell`
如果能进入这个那就大功告成
```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.0
      /_/
         
Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_121)
Type in expressions to have them evaluated.
Type :help for more information.
```

顺带再说一句，由于我们只用作单机测试用嘛，所以有不少东西都没配，也就是说会出现各种不痛不痒(大概？毕竟我测试程序能跑)的error和一些warning。解决的方法貌似得涉及到源码编译，蛮麻烦的，所以我比较懒就先不弄了，反正是能跑起来了。如果以后有需要再研究。