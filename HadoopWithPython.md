# Hadoop with Python
## 1 HDFS概述
### 1.1 HDFS的结构
HDFS由两个进程组成:
- NameNode 
- DataNode

NameNode(唯一)存放文件系统的元数据，DataNode(一个以上)存放构成文件的块(一个文件由许多块组成，一个文件的块可能存放在不同的机器上)。NameNode和DataNode可以在一台机器上运行，但是一般的，HDFS集群由一个专用服务器运行NameNode进程，其他成千上万台机器运行DataNode进程。

NameNode是HDFS中最重要的部分，为快速的访问信息，NameNode将整个元数据结构存放在内存中，它存放了整个文件系统的元数据:
- 文件名
- 文件权限
- 每个文件的块的位置

另外的，NameNode还有追踪块的复制因子的作用，以避免机器故障导致的数据丢失。由于NameNode is a single porint of failures(不知道怎么翻译好)，所以一个二级NameNode可以用来生成初级NameNode的内存结构快照，以减少NameNode故障时的数据丢失风险。
## 2 MapReduce with Python
MapReduce将工作分为独立任务和执行任务，是允许在平行的集群中处理和生成大数据量的一个编程模型。每个MapReduce程序两次将输入数据元素转换为输出数据元素，一次在map阶段，一次在reduce阶段。

### 2.1 数据流
MapReduce框架由三个主要阶段组成：
- map
- shuffle and sort
- reduce

在这个部分将会详细的讲述每个阶段。
#### 2.1.1 Map阶段
MapReduce应用的第一个阶段是map阶段。在map阶段中，一个被称作mapper的函数处理一系列的**key-value**对。mapper会顺序的逐个处理每个输入的**key-value**对，然后生成0个或多个输出**key-value**对。例如一个split程序写出的mapper能够做到下面的事情:

 Input |Mapper  | Output
-------|--------|:-----:
|||"zhang"
 "zhang xin nan" | ------------------->|"xin"
|||"nan"

#### 2.1.2 Shuffle and Sort
MapReduce的第二个阶段是shuffle and sort。mappers完成后，map极端生成的中间结果会被移动到reducers，这个移动的过程被称作shuffling。
shuffling过程由一个划分函数处理，被称作划分器(partitioner)。划分器的作用是控制从mappers到reducers的key-value对的“流”。对给定mapper输出的key和reducers的数量，划分器会返回所需reducer的索引。划分器会确保所有具有相同key的values被送到同一个reducer。然后是sort过程，在这些values被送到reducer之前，会由Hadoop框架排序。

#### 2.1.3 Reduce
Reducer是一个具有迭代values功能的函数，它对每个key聚合其的values，生成0个或多个key-value对。例如：

 Input |Reducer  | Output
:-----:|--------|:-----:
"cat": {[3, 4, 1]} | ------------------->|"cat: {8}"

### 2.2 Hadoop流
Hadoop流允许任何像mapper或者reducer的可执行程序创建MapReduce的过程。这是Hadoop with Python的关键，当然，因为hadoop流的存在，像shell脚本或者其他一些编程语言都可以用来写mapper或者reducer。
#### 2.2.1 工作原理
Mapper和reducer都是从标准输入(stdin)中一行行读取，或
写到标准输出(stdout)的可执行程序。

所以在使用其他语言写mapper和reducer的时候，只要使用stdin和stdout来操作数据的输入与输出，就能在hadoop上跑起来了。

#### 2.2.2 Python示例
在这里，使用Python作为编程语言，来编写了实现WordCount功能的mapper.py和reducer.py。

Example 1. mapper.py
```
#!/usr/bin/env python
import sys

# Read each line from stdin
for line in sys.stdin:

	# Get the words in each line
	words = line.split()
    
# Generate the count for each word
for word in words:

    # Write the key-value pair to stdout to be processed by
    # the reducer.
    # The key is anything before the first tab character and the
    # value is anything after the first tab character.
	print('{0}\t{1}'.format(word, 1))
```

Example 2. reducer.py
```
#!/usr/bin/env python
import sys

curr_word = None
curr_count = 0

# Process each key-value pair from the mapper
for line in sys.stdin:

	# Get the key and value from the current line
	word, count = line.split('\t')
    
	# Convert the count to an int
	count = int(count)
    
    # If the current word is the same as the previous word,
    # increment its count, otherwise print the words count
    # to stdout
    if word == curr_word:
		curr_count += count
	else:
    
        # Write word and its number of occurrences as a key-value
        # pair to stdout
		if curr_word:
            print '{0}\t{1}'.format(curr_word, curr_count)
            curr_word = word
			curr_count = count
            
# Output the count for the last word
if curr_word == word:
	print '{0}\t{1}'.format(curr_word, curr_count)
```

在执行代码以前，有一个很关键的步骤，在之前搭建hadoop环境实的测试示例上也说过，就是权限问题，即需要确保mapper.py和reducer.py这两个文件具有执行权限，下面这行命令可以做到这一点：

`$ chmod a+x mapper.py reducer.py`

还有一点就是，要确保文件(mapper.py和reducer.py)的第一行，包含Python的路径，如示例中的`#!/usr/bin/env python`
，如果发现运行时出现错误，需要将这一行代码替换为本地系统的python路径。

在将这个Python程序作为MapReduce任务前，可以使用**echo**和**sort**命令在本地的shell里进行测试。在这里强烈的建议，对于所有的程序，在放在Hadoop集群上运行前，一定要先做本地测试，该程序的测试命令如下：
```
$ echo 'zxn is fool zxn is smart' | ./mapper.py | sort -t 1 | ./reducer.py
```
程序执行后会输出：
```
fool     1
is       2
smart    1
zxn      2
```

测试成功后，就可以放在集群上跑了，其命令如下：
```
$ $HADOOP_HOME/bin/hadoop jar 
  $HADOOP_HOME/mapred/contrib/streaming/hadoop-streaming*.jar \
-files mapper.py,reducer.py \
-mapper mapper.py \
-reducer reducer.py \
-input /user/hduser/input.txt -output /user/hduser/output
```
#### 2.2.3 mrjob

在这里介绍一个框架，它就是mrjob，是一个Python MapReduce库。使用它会使Hadoop with Python更简单。

mrjob的安装:

`$ pip install mrjob`

或者从git上clone下来后:

`$ python setup.py install`

**WordCount in mrjob示例**

Example3. word_count.py
```
from mrjob.job import MRJob

class MRWordCount(MRJob):
	def mapper(self, _, line):
	for word in line.split():
		yield(word, 1)
        
	def reducer(self, word, counts):
		yield(word, sum(counts))
        
if __name__ == '__main__':
MRWordCount.run()
```

想要本地执行这个程序的话，只需要:

`$ python word_count.py input.txt`

默认的，mrjob会将结果写到stdout.

如果输入文件有多个的话，可以直接这样做:

`$ python word_count.py input1.txt input2.txt input3.txt`

mrjob也可以通过stdin来进行输入:

`$ python word_count.py < input.txt`

默认的，mrjob是在本地执行，便于调试。如果要改变运行方式的话，需要指定 -r/--runner 选项。可供选择的项一共4个：

command|detail|
------|------
-r inline|(Default) Run in a single Python process
-r local|Run locally in a few subprocesses simulating some Hadoop features
-r hadoop| Run on a Hadoop cluster
-r emr |Run on Amazon Elastic Map Reduce (EMR)

这里再举一更复杂的例子，用于计算员工最高年薪和总薪酬。数据来源于2014年巴尔迪莫的**薪水数据**。

Example 4. top_salary.py
```
from mrjob.job import MRJob
from mrjob.step import MRStep
import csv

cols = 'Name,JobTitle,AgencyID,Agency,HireDate,AnnualSalary,Gross
Pay'.split(',')

class salarymax(MRJob):
	def mapper(self, _, line):
		# Convert each line into a dictionary
		row = dict(zip(cols, [ a.strip() for a in csv.reader([line]).next()]))
		# Yield the salary
    	yield 'salary', (float(row['AnnualSalary'][1:]), line)
    
		# Yield the gross pay
		try:
			yield 'gross', (float(row['GrossPay'][1:]), line)
		except ValueError:
			self.increment_counter('warn', 'missing gross', 1)

	def reducer(self, key, values):
		topten = []
		
        # For 'salary' and 'gross' compute the top 10
		for p in values:
            topten.append(p)
            topten.sort()
            topten = topten[-10:]
        
		for p in topten:
			yield key, p

	combiner = reducer

if __name__ == '__main__':
	salarymax.run()
```
使用一下命令在集群中运行：

`$ python top_salary.py -r hadoop hdfs:///user/hduser/input/salaries.csv`

关于mrjob更详细的介绍可以查看[官方文档](https://pythonhosted.org/mrjob/)或者访问[官方github](https://github.com/Yelp/mrjob).







