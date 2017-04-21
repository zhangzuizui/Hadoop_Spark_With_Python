# Hadoop服务器搭建
我用的系统版本是ubuntu16.04，若安装过程中出现其他问题，谷歌一下即可解决。
## 1 Hadoop启动模式
Hadoop集群有三种启动模式：
- 单机模式：默认情况下运行为一个单独机器上的独立Java进程，主要用于调试环境
- 伪分布模式：在单个机器上模拟成分布式多节点环境，每一个Hadoop守护进程都作为一个独立的Java进程运行
- 完全分布式模式：真实的生产环境，搭建在完全分布式的集群环境

## 2 Hadoop单机模式安装
### 2.1 用户及用户组
在搭建hadoop开发环境之前，需要先添加一个用来运行Hadoop进程的用户和用户组
#### 2.1.1 添加用户和用户组
创建用户hadoop

`$ sudo adduser hadoop`

并按照提示输入hadoop用户的密码
#### 2.1.2 添加sudo权限
将hadoop用户添加进sudo用户组

`$ sudo usermod -G sudo hadoop`

### 2.2 安装及配置依赖的软件包
#### 2.2.1 相关依赖包安装
```
$ sudo apt-get update 
$ sudo apt-get install openssh-server rsync
$ sudo service ssh restart
$ sudo apt-get install openjdk-8-jdk
```
#### 2.2.2 配置ssh免密码登录
切换到hadoop用户，需要输入添加hadoop用户时配置的密码。后续步骤都将在hadoop用户的环境中执行。

`$ su -l hadoop`

在/home/hadoop目录下执行
```
$ ssh-keygen -t rsa	#一路回车
$ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
$ chmod 600 .ssh/authorized_keys
```
验证登录本机是否还需要密码，设置成功后除了第一次，以后都不要密码了，便于使用。

`$ ssh localhost`

### 2.3 下载并安装Hadoop
在hadoop用户登录的环境中进行下列操作：
#### 2.3.1 下载hadoop2.8.0
这是2.8.0的官方提供的一个镜像源，下不下来的话自己百度一个下吧。下2.7.3版后面的安装过程应该也差不多。
```
$  su hadoop
$  hadoop
$  sudo wget http://www-eu.apache.org/dist/hadoop/common/hadoop-2.8.0/hadoop-2.8.0-src.tar.gz
```
#### 2.3.2 解压并安装
```
$ sudo tar zxvf hadoop-2.8.0.tar.gz
$ sudo mv hadoop-2.8.0 /usr/local/hadoop
$ sudo chmod 777 /usr/local/hadoop
```
#### 2.3.3 配置hadoop

`$ vim /home/hadoop/.bashrc`

在文件末尾添加下列内容：
```
#HADOOP START
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_INSTALL=/usr/local/hadoop
export PATH=$PATH:$HADOOP_INSTALL/bin
export PATH=$PATH:$HADOOP_INSTALL/sbin
export HADOOP_MAPRED_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_HOME=$HADOOP_INSTALL
export HADOOP_HDFS_HOME=$HADOOP_INSTALL
export YARN_HOME=$HADOOP_INSTALL
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_INSTALL/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_INSTALL/lib"
#HADOOP END
```
保存退出后，激活应用新的环境变量

`$ source ~/.bashrc`

到这里Hadoop的单机模式就安装完成了。
### 2.4 测试
#### 2.4.1 创建输入的数据
采用/etc/protocols作为测试用文件
```
$ cd /usr/local/hadoop
$ sudo mkdir input
$ sudo cp /etc/protocols ./input
```
#### 2.4.2 执行Hadoop WordCount应用（词频统计）
`$ bin/hadoop jar share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.8.0-sources.jar org.apache.hadoop.examples.WordCount input output`

若出现jar什么什么的问题，需要切换一下当前目录：
`$ cd $HADOOP_INSTALL`

#### 2.4.3 查看生成的单词统计数据
`$ cat output/*`
## 3 Hadoop伪分布模式配置部署
基于之前搭建的hadoop单机模式，通过相关操作可以搭建一个只有一台设备的'分布式'集群，称之为'伪分布式'。
### 3.1 相关配置文件修改
#### 3.1.1 修改core-site.xml:
`$ sudo vim /usr/local/hadoop/etc/hadoop/core-site.xml`
```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/hadoop/tmp</value>
   </property>
</configuration>
```
常用配置项说明：

- **fs.default.name**这是一个描述集群中NameNode结点的URI(包括协议、主机名称、端口号)，集群里面的每一台机器都需要知道NameNode的地址。DataNode结点会先在NameNode上注册，这样它们的数据才可以被使用。独立的客户端程序通过这个URI跟DataNode交互，以取得文件的块列表。
- **hadoop.tmp.dir** 是hadoop文件系统依赖的基础配置，很多路径都依赖它。如果hdfs-site.xml中不配置namenode和datanode的存放位置，默认就放在**/tmp/hadoop-${user.name}**这个路径中。

更多说明请参考[core-default.xml](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/core-default.xml)，包含配置文件所有配置项的说明和默认值。
#### 3.1.2 修改hdfs-site.xml:
`$ sudo vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml`

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
常用配置项说明：

- **dfs.replication**它决定着系统里面的文件块的数据备份个数。对于一个实际的应用，它应该被设为3（这个数字并没有上限，但更多的备份可能并没有作用，而且会占用更多的空间）。少于三个的备份，可能会影响到数据的可靠性(系统故障时，也许会造成数据丢失)。
- **dfs.data.dir**这是DataNode结点被指定要存储数据的本地文件系统路径。DataNode结点上的这个路径没有必要完全相同，因为每台机器的环境很可能是不一样的。但如果每台机器上的这个路径都是统一配置的话，会使工作变得简单一些。默认的情况下，它的值为**file://${hadoop.tmp.dir}/dfs/data**这个路径只能用于测试的目的，因为它很可能会丢失掉一些数据。所以这个值最好还是被覆盖。
- **dfs.name.dir**这是NameNode结点存储hadoop文件系统信息的本地系统路径。这个值只对NameNode有效，DataNode并不需要使用到它。上面对于/temp类型的警告，同样也适用于这里。在实际应用中，它最好被覆盖掉。

更多说明请参考[hdfs-default.xml](http://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)，包含配置文件所有配置项的说明和默认值。
#### 3.1.3 修改mapred-site.xml:
```
$ sudo cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml
$ sudo vim /usr/local/hadoop/etc/hadoop/mapred-site.xml
```
```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
常用配置项说明：

- **mapred.job.trackerJobTracker**的主机（或者IP）和端口。

更多说明请参考[mapred-default.xml](http://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)，包含配置文件所有配置项的说明和默认值。
#### 3.1.4 修改yarn-site.xml
`$ sudo vim /usr/local/hadoop/etc/hadoop/yarn-site.xml`

```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

</configuration>
```
常用配置项说明：

- **yarn.nodemanager.aux-services**通过该配置，用户可以自定义一些服务。

更多说明请参考[yarn-default.xml](http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)，包含配置文件所有配置项的说明和默认值。
#### 3.1.5 修改hadoop-env.sh:
`$ sudo vim /usr/local/hadoop/etc/hadoop/hadoop-env.sh`
修改JAVA_HOME如下:
```
#The java implementation to use.
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```
这样简单的伪分布式格式就配置好了。
### 3.2 格式化HDFS文件系统
在使用hadoop前，必须格式化一个全新的HDFS安装，通过创建存储目录和NameNode持久化数据结构的初始版本，格式化过程创建了一个空的文件系统。由于NameNode管理文件系统的元数据，而DataNode可以动态的加入或离开集群，因此这个格式化过程并不涉及DataNode。同理，用户也无需关注文件系统的规模。集群中DataNode的数量决定着文件系统的规模。DataNode可以在文件系统格式化之后的很长一段时间内按需增加。

格式化文件系统的指令为:

`$ hadoop namenode -format`

会输出如下信息，表示格式化HDFS成功:
```
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

INFO namenode.NameNode: STARTUP_MSG:
/************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = [你的主机名]/[你的ip]
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 2.6.0
...
...
INFO util.GSet: Computing capacity for map NameNodeRetryCache
INFO util.GSet: VM type       = 64-bit
INFO util.GSet: 0.029999999329447746% max memory 889 MB = 273.1 KB
INFO util.GSet: capacity      = 2^15 = 32768 entries
INFO namenode.NNConf: ACLs enabled? false
INFO namenode.NNConf: XAttrs enabled? true
INFO namenode.NNConf: Maximum size of an xattr: 16384
INFO namenode.FSImage: Allocated new BlockPoolId: BP-549895748-192.168.42.3-1489569976471
INFO common.Storage: Storage directory /home/hadoop/hadop2.6-tmp/dfs/name has been successfully formatted.
INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
NFO util.ExitUtil: Exiting with status 0
INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at [你的主机名]//[你的ip]
************************************************************/
```
### 3.3 Hadoop集群启动
#### 3.3.1 启动hdfs守护进程
分别启动NameNode和DataNode

`$ start-dfs.sh`

输出如下（可以看出分别启动了namenode, datanode, secondarynamenode，因为我们没有配置secondarynamenode，所以地址为0.0.0.0）：
```
Starting namenodes on []
hadoop@localhost's password:
localhost: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-G470.out
hadoop@localhost's password:
localhost: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-G470.out
localhost: OpenJDK 64-Bit Server VM warning: You have loaded library /usr/local/hadoop/lib/native/libhadoop.so.1.0.0 which might have disabled stack guard. The VM will try to fix the stack guard now.
localhost: It's highly recommended that you fix the library with 'execstack -c <libfile>', or link it with '-z noexecstack'.
Starting secondary namenodes [0.0.0.0]
hadoop@0.0.0.0's password:
0.0.0.0: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-secondarynamenode-G470.out
```
**注意**这里有可能会出现这个警告:

`“Unable to load native-hadoop library for your platform”` 

这是因为(有可能)我们的系统是64位，而hadoop-lib是32位的，其实是没有什么影响的，如果想屏蔽这个警告的话，此时需要进入hadoop-env.sh，修改HADOOP_OPTS:

`export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=/usr/local/hadoop/lib/native"`

### 3.4 启动yarn
启动ResourceManager和NodeManager:

`$ start-yarn.sh`

### 3.5 检查是否运行成功
打开浏览器
- 输入：http://localhost:8088 进入ResourceManager管理页面
- 输入：http://localhost:50070 进入HDFS页面

**可能出现的问题及调试方法：**
启动伪分布后，如果活跃节点显示为零，说明伪分布没有真正的启动。原因是有的时候数据结构出现问题会造成无法启动datanode。如果使用**hadoop namenode -format**重新格式化仍然无法正常启动，原因是**/tmp**中的文件没有清除，则需要先清除**/tmp/hadoop/\***再执行格式化，即可解决hadoop datanode无法启动的问题。具体步骤如下所示：
```
# 删除hadoop:/tmp
$ hadoop fs -rmr /tmp
# 停止hadoop
$ stop-all.sh
# 删除/tmp/hadoop*
$ rm -rf /tmp/hadoop*
# 格式化
$ hadoop namenode -format
# 启动hadoop
$ start-all.sh
```
### 3.6 测试
测试还是使用WordCount。不同的是，这次使用伪分布式进行测试，将会使用到hdfs，因此需要将文件拷贝到hdfs上去。
#### 3.6.1 创建相关文件夹:
```
$ hadoop fs -mkdir /user
$ hadoop fs -mkdir /user/hadoop
$ hadoop fs -mkdir /user/hadoop/input
```
#### 3.6.2 创建输入的数据
采用/etc/protocols文件作为测试，先将文件拷贝到hdfs上:
`$ hadoop fs -put /etc/protocols /user/hadoop/input`
**注意**关于hadoop的更多shell命令可以看[官方文档](https://hadoop.apache.org/docs/r1.0.4/cn/hdfs_shell.html)，有官方中文翻译，很开心。

#### 3.6.3 执行Hadoop WordCount应用(词频统计)
```
# 如果存在上一次测试生成的output，由于hadoop的安全机制，直接运行可能会报错，所以请手动删除上一次生成的output文件夹
#注意，这里也挺烦的，就是执行这个示例必须先cd一下，hadoop的安装目录，因为不这样做的话jar指令会出问题
$ cd $HADOOP_INSTALL
$ hadoop jar share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.8.0-sources.jar org.apache.hadoop.examples.WordCount /user/hadoop/input output
```
#### 3.6.4 查看生成的单词统计数据
`$ hadoop fs -cat /user/hadoop/output/*`
### 3.7 关闭服务
```
$ stop-dfs.sh
$ stop-yarn.sh
```
## 参考文档
- http://hadoop.apache.org/docs/r2.7.3/ 对于2.7.3版本的hadoop搭建可在当前文档的基础上参考这个
- http://www.cnblogs.com/taichu/p/5264185.html
- 实验楼hadoop搭建教程


