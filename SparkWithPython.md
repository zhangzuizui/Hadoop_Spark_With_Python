# Spark With Python
## 1. Spark简介
跟Hadoop相似，Spark是一个集群计算框架。但是因为它基于内存的特性，所以程序在Spark上的运行速度能比在Hadoop上快一百倍。Spark应用由一个通过集群来控制并行操作执行的驱动程序组成。Spark提供了名为**弹性分布式数据集(Resilient Distributed Datasets, RDDs)** 的数据抽象，RDDs是通过集群的节点划分的，可以进行并行操作的元素集合。
## 2. PySpark
### 2.1 简介
PySpark是Spark的Python API，PySpark允许Spark应用程序通过交互式shell或者Python程序创建。需要注意的是，初始状态下的PySpark并没有被添加到PYTHONPATH，所以在编辑器或者Python的解释器里面是不能`import pyspark`的，需要将下面的内容添加到.bashrc中。

`$ vim ~/.bashrc`
```
#这个就是spark安装路径，如果是一直在跟着我的文档走那么就是这个路径了
export SPARK_HOME=/usr/local/spark 

export PYTHONPATH=$SPARK_HOME/python/:$PYTHONPATH
export PYTHONPATH=$SPARK_HOME/python/lib/py4j-0.10-src.zip:$PYTHONPATH

#如果还不行的话添加一行这个
export PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/build:$PYTHONPATH
```
每次添加完用户配置都要

`source ~/.bashrc`

使其生效，如果是减少了某个配置项的话，需要执行一个重新执行所有系统脚本的命令，我忘了是啥了，没谷歌到，简单点直接注销吧。
### 2.2 WordCount in PySpark
下面的例子中，在PySpark里实现了WordCount. 这里假设有一个数据文件， input.txt 已经被读取到HDFS中，在/user/hduser/input下，同时输出将被放在/user/hduser/output.

Example 1. python/Spark/word_count.py

```
from pyspark import SparkContext

def main():
	
    #创建一个SparkContext对象，这个对象
    #告诉Spark如何以及去哪里访问集群
	sc = SparkContext(appName='SparkWordCount')
    
    #使用SparkContext从HDFS中读取文件
    #并将文件存放在input_file变量中
	input_file = sc.textFile('/user/hduser/input/input.txt')
    
    #这里Spark会自动在多台机器上并行的进行下面的转换过程
	counts = input_file.flatMap(lambda line: line.split()) \
							.map(lambda word: (word, 1)) \
							.reduceByKey(lambda a, b: a + b)
                            
    #将结果存放到HDFS
	counts.saveAsTextFile('/user/hduser/output')
    
    #关闭SparkContext
	sc.stop()
    
if __name__ == '__main__':
	main()
```
通过将文件名传到spark-submit脚本中来执行这个Spark应用:

`$ spark-submit --master local word_count.py`

当这个程序执行的时候，一系列的文本会被打印到控制台上，最终程序的运行结果能够在 /user/hduser/output/part-00000找到。
不论用什么代码执行Spark，其应用程序都必须创建一个SparkContext对象，这个对象告诉Spark如何执行，以及在哪里访问集群，**master**属性是集群的URL，决定Spark应用的运行位置，常用的master值有这些：

values|detail|
--|--
local|Run Spark with one worker thread.
local[n]|Run Spark with n worker threads.
spark://HOST:PORT|Connect to a Spark standalone cluster.
mesos://HOST:PORT|Connect to a Mesos cluster.

master属性也能在编程时设置，只需要:

`sc = SparkContext(master='local[4]')`

执行这样的程序时，需要将其递交给spark-submit脚本。

`$ spark-submit --master local spark_app.py`

更多参数可以运行`pyspark --help`查看。

### 2.3 关于交互式shell

当Spark shell启动时，SparkContext就会被创建，存放在变量sc中。交互式shell的master可以在启动shell时，使用`--master`参数设置，交互式shell的启动命令举例:

`$ pyspark --master local[4]`

对于完整的选项列表，可以查看`pyspark --help`.
## 3 RDDs
RDDs是Spark的基础编程抽象，是跨机器分区(partitioned across machines)的不可变的数据集合，能够操作并行执行的元素。RDDs可以以多种方式构建：通过并行化现有的Python集合，通过引用外部存储系统（例如HDFS）中的文件，或通过对现有RDDs应用转换。
### 3.1 从集合创建RDDs
RDDs可以通过调用Spark中的Context.parallelize()方法从Python集合创建，元素的集合被复制组成一个可以被并行操作的分布式数据集。下面的例子是从一个Python list创建并行集合：
```
>>> data = [1, 2, 3, 4, 5]
>>> rdd = sc.parallelize(data)
>>> rdd.glom().collect()
...
[[1, 2, 3, 4, 5]]
```
如果遇到说sc没有定义之类的错误，那么加上一行这个，大概是因为之前创建了没有关之类的吧，我也不知道什么情况会出现这个。

`sc = SparkContext.getOrCreate()`

RDD.glom()方法返回一个包括了每个分区中所有元素的列表，RDD.collect()方法将所有元素传给驱动节点，这个[[1, 2, 3, 4, 5]]是列表中的原始集合。

要想指定RDD创建时的分区数，可以给parallelize()方法传入第二个参数。下面的例子从同样的Python集合中创建了一个RDD，不同的是这次建立了4个分区：
```
>>> rdd = sc.parallelize(data, 4)
>>> rdd.glom().collect()
...
[[1], [2], [3], [4, 5]]
```
需要注意的是，如果在进入pyspark shell的时候，直接使用命令

`$ pyspark --master local[4]`

那么上面的示例及时不创建4个分区，输出结果也是[[1], [2], [3], [4, 5]].

### 3.2 从外部资源创建RDDs
使用SparkContext.textFile()方法能从文件创建RDD，Spark可以读取本地文件系统内的文件，Hadoop， Amazon S3等支持的所有存储源都可。Spark支持文本文件，SequenceFiles，任何其他 Hadoop 输入格式，目录，压缩文件和通配符，例如 _my/directory/\*.txt_. 下面的例子展示了从本地文件系统的一个分拣中创建分布式数据集的过程：
```
>>> distFile = sc.textFile('data.txt')
>>> distFile.glom().collect()
...
[[u'jack be nimble', u'jack be quick', u'jack jumped over the
candlestick']]
```
### 3.3 RDD操作
RDDs支持两个类型的操作：转换(transformations)和行为(actions)。**转换**从已存在的数据中，创建新的数据集，**动作**是在数据集上进行计算，并将结果返回给驱动程序。
#### 3.3.1 transformations
**转换**是惰性的，也就是说其结果不会马上计算，而是由Spark记住应用于一个基础数据集的所有转换。当一个**动作**需要返回给驱动程序结果的时候，**转换**才会被计算。这使Spark的操作具有高效性，并且只会在一个**动作**之前传输**转换**的结果。关于**转换**的详细文档可以参考[Sparks' Python RDD API doc](http://bit.ly/1MtRa2t).

**map:** map(_func_)方法通过对资源内的每个元素应用一个函数 _func_，返回新的RDD，下面的例子里资源里的每个元素都被乘了2：
```
>>> data = [1, 2, 3, 4, 5, 6]
>>> rdd = sc.parallelize(data)
>>> map_result = rdd.map(lambda x: x * 2)
>>> map_result.collect()
[2, 4, 6, 8, 10, 12]
```
**filter:** filter(_func_)函数能够返回一个新的RDD容器，新容器的元素对于提供的函数，返回值为ture(哎呀反正就是一个过滤器过滤一下，翻译起来太别扭)。
```
>>> data = [1, 2, 3, 4, 5, 6]
>>> filter_result = rdd.filter(lambda x: x % 2 == 0)
>>> filter_result.collect()
[2, 4, 6]
```
**distinct:** 就是返回一个不带重复元素的新RDD容器，例子如下：
```
>>> data = [1, 2, 3, 2, 4, 1]
>>> rdd = sc.parallelize(data)
>>> distinct_result = rdd.distinct()
>>> distinct_result.collect()
[4, 1, 2, 3]
```
**flatMap:** 和map()很像，但是结果是一个扁平化的版本(flattened version)，不懂具体啥意思，看一个对比示例意会一下吧：
```
>>> data = [1, 2, 3, 4]
>>> rdd = sc.parallelize(data)
>>> map = rdd.map(lambda x: [x, pow(x,2)])
>>> map.collect()
[[1, 1], [2, 4], [3, 9], [4, 16]]

>>> rdd = sc.parallelize()
>>> flat_map = rdd.flatMap(lambda x: [x, pow(x,2)])
>>> flat_map.collect()
[1, 1, 2, 4, 3, 9, 4, 16]
```
大概就是无视了匿名函数里面的那个list框框？

#### 3.3.2 Actions
详细的东西看这里[Spark’s Python RDD API
doc](http://bit.ly/1MtRa2t).

**reduce:** RDD里的reduce()方法的话，跟python的reduce是同一个东西，会一直累积，举例如下：
```
>>> data = [1, 2, 3]
>>> rdd = sc.parallelize(data)
>>> rdd.reduce(lambda a, b: a * b)
6
```
**take:** take(_n_)方法返回一个RDD前n个元素的数组。例子如下：
```
>>> data = [1, 2, 3]
>>> rdd = sc.parallelize(data)
>>> rdd.take(2)
[1, 2]
```

**collect:** collect()方法一一个数组的形式返回RDD内的所有元素。例子如下：
```
>>> data = [1, 2, 3, 4, 5]
>>> rdd = sc.parallelize(data)
>>> rdd.collect()
[1, 2, 3, 4, 5]
```
**注意！！！** 在太大的数据集上使用collect()会导致驱动器耗尽内存，所以想要检查很大的RDDs，take()和collect()方法可以一起用，以输出top n个元素。

`>>> rdd.take(100).collect()`

**takeOIrdered:** takeOrdered(_n_, _key=func_)返回RDD的前 _n_ 个元素,返回的顺序为以他们的自然顺序(从大到小)，或通过一个指定函数后的排序顺序。例子如下：
```
>>> data = [6,1,5,2,4,3]
>>> rdd = sc.parallelize(data)
>>> rdd.takeOrdered(4, lambda s: -s)
[6, 5, 4, 3]
```
默认的，每当一个**动作**执行时，**转换**都有可能被重新计算，这时Spark能够高效地利用内存，但是如果同样的**转换**不断的被执行的话，会占用更多的进程资源。为确保一个**转换**只计算一次，可使用RDD.cache()方法让一个resulting RDD驻留在内存中。
#### 3.3.3 RDD工作流程
通常RDD按下述步骤操作：
1. 从一个数据源创建RDD
2. 对RDD进行**转换**
3. 对RDD进行**动作**

下例使用这个工作流程来计算一个文件中的字符数：
```
>>> lines = sc.textFile('data.txt')
>>> line_lengths = lines.map(lambda x: len(x))
>>> document_length = line_lengths.reduce(lambda x,y: x+y)
>>> print(document_length)
```

第一条语句是从外部文件 _data.txt_ 创建了一个RDD，这个文件不会在这个时候加载，变量**lines**仅仅是一个指向外部文件的指针。第二条语句在基础RDD上使用map()方法执行了一条**转换**来计算每行的字符数。变量**line_lengths**由于**转换**的惰性，不会被马上计算。最后调用reduce()方法执行**动作**。这个时候Spark将计算分为多个任务在不同的机器上运行，每台机器都在它的本地数据上运行map和reduce，只将结果返回给驱动程序

如果应用可能再次使用**line_lengths**的话，最好将map的**转换**结果“固定住”(persist)以确保map不会被再次计算。下面这行代码能够在第一次计算后将line_lengths存进内存：

`>>> line_lengths.persist()`

## 4 来一个文本搜索的例子
这个文本搜索程序的目的是搜索与给定string相匹配的电影标题，电影数据是[groupLens](http://grouplens.org/datasets/movielens/)数据集，这个应用程序预期被存在HDFS中的/user/
hduser/input/movies。

Example2. text_search.py
```
from pyspark import SparkContext
import re
import sys

def main():

    # Insure a search term was supplied at the command line
    if len(sys.argv) != 2:
    	sys.stderr.write('Usage: {} <search_term>'.format(sys.argv[0]))
    	sys.exit()
        
    # Create the SparkContext
    sc = SparkContext(appName='SparkWordCount')
    
    # Broadcast the requested term
    requested_movie = sc.broadcast(sys.argv[1])
    
    # Load the input file
    source_file = sc.textFile('/user/hduser/input/movies')
    
    # Get the movie title from the second fields
    titles = source_file.map(lambda line: line.split('|')[1])
    
    # Create a map of the normalized title to the raw title
    normalized_title = titles.map(lambda title: (re.sub(r'\s*\
    (\d{4}\)','', title).lower(), title))
    
    # Find all movies matching the requested_movie
    matches = normalized_title.filter(lambda x: reques  ted_movie.value in x[0])
    
    # Collect all the matching titles
    matching_titles = matches.map(lambda x: x[1]).distinct().collect()
    
    # Display the result
    print('{} Matching titles found:'.format(len(matching_titles)))    
    for title in matching_titles:
    	print(title)
    
    sc.stop()
    
if __name__ == '__main__':
	main()
```
一个执行示例如下：
```
$ spark-submit text_search.py gold
...
6 Matching titles found:
GoldenEye (1995)
On Golden Pond (1981)
Ulee's Gold (1997)
City Slickers II: The Legend of Curly's Gold (1994)
Golden Earrings (1947)
Gold Diggers: The Secret of Bear Mountain (1995)
...
```
因为计算**转换**是一个开销很大的操作，Spark可以把标准化标题(normalized_titles)的结果缓存到内存里以加快未来的搜索速度。在上面的例子中，为了将标准化标题读取到内存里，使用了cache()方法:

`normalized_title.cache()`