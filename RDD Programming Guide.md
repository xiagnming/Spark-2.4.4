# Overview
从较高级别上来说，每个Spark程序都包含一个驱动程序，该驱动程序运行用户的主要功能，并在集群上执行各种并行操作。 
Spark提供的主要抽象是弹性分布式数据集（RDD），它是一个集群节点分区的元素集合，可以并行操作.
RDDs是通过从Hadoop文件系统（或任何其他Hadoop支持的文件系统） 或现有的Scala集合中进行转换得到。用户还可以把RDD缓存在内存中，
以使其在并行操作中有效地重用。最后，RDD还可以从节点故障中自动恢复。

Spark中的第二个抽象是可以在并行操作中使用共享变量。默认情况下，当Spark作为一组任务在不同节点上并行运行一个函数式，
它会将函数中使用的每个变量的副本传送给每个任务。有时，需要在任务之间或任务与驱动程序之间共享变量。Spark支持两种类型的共享变量：广播变量（
可用于所有节点上的内存中的缓存值）和累加器（是仅添加到其中的变量，如计数器和）。本指南以Spark支持的语言展示了spark的每种功能。如果您想通过交互式的方式
启动spark，最简单的方法就是Scala shell：bin/spark-shell，或者是Python：bin/pyspark

# Linking with Spark

Spark 2.4.4适用于Python 2.7+或Python 3.4+。 它可以使用标准的CPython解释器，因此可以使用NumPy之类的C库。 它还适用于PyPy 2.3+。

在Spark 2.2.0中删除了对Python 2.6的支持。

Python中的Spark应用程序既可以在运行时使用包含Spark的bin / spark-submit脚本运行，也可以将其包含在setup.py中，如下所示：
```python
install_requires=[
    'pyspark=={site.SPARK_VERSION}'
]
```
要在Python中运行Spark应用程序而无需pip安装PySpark，请使用位于Spark目录中的bin / spark-submit脚本。 
该脚本将加载Spark的Java / Scala库，并允许您将应用程序提交到集群。 您还可以使用bin / pyspark启动交互式Python Shell。

如果您希望访问HDFS数据，您需要使用PySpark构建来链接到您的HDFS版本。在Spark的主页上还提供了一些预构建的软件包
[Prebuilt packages](https://spark.apache.org/downloads.html)，用于常见的HDFS版本。

最后，你需要引入Spark的类到你的项目中。
```python
from pyspark import SparkContext, SparkConf
```
PySpark在驱动程序和工作程序中都需要相同的Python版本。它使用PATH中的默认python版本，例如，您可以指定PYSPARK python要使用的python版本:
```python
$ PYSPARK_PYTHON=python3.4 bin/pyspark
$ PYSPARK_PYTHON=/opt/pypy-2.5/bin/pypy bin/spark-submit examples/src/main/python/pi.py
```

# Initializing Spark

Spark程序必须做的第一件事是创建一个SparkContext对象，它告诉Spark如何访问集群。
要创建SparkContext，首先需要构建一个包含应用程序信息的SparkConf对象。
```python
conf  =SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```
ppName参数是您的应用程序在群集UI上显示的名称。 master是一个Spark，Mesos或YARN群集URL，或一个特殊的“local”字符串，以本地模式运行。 
实际上，当在集群上运行时，您将不希望对程序中的母版进行硬编码，而是希望通过spark-submit启动应用程序并在其中接收它。 
但是，对于本地测试和单元测试，您可以传递“ local”以在内部运行Spark。

## Using the shell

在PySpark shell中，已经在名为sc的变量中为您创建了一个特殊的可识别解释程序的SparkContext。 制作自己的SparkContext将不起作用。 
您可以使用--master参数设置上下文连接的主机，也可以通过将逗号分隔的列表传递给--py-files，将Python .zip，.egg或.py文件添加到运行时路径。 
您还可以通过在--packages参数中提供逗号分隔的Maven坐标列表，从而将依赖项（例如Spark Packages）添加到Shell会话中。 
可以存在依赖项的任何其他存储库（例如Sonatype）都可以传递给--repositories参数。 
必要时，必须使用pip手动安装Spark软件包具有的所有Python依赖项（在该软件包的requirements.txt中列出）。
```python
$ ./bin/pyspark --master local[4]
```
或者添加code.py到搜索路径：
```python
$ ./bin/pyspark --,aster local[4] --py-files code.py
```
要获得完整的选项列表，请运行pyspark——help。在内部，pyspark调用更一般的spark-submit脚本。

同样，可以在增强的Python解释器IPython中启动PySpark Shell。 PySpark可与IPython 1.0.0及更高版本一起使用。 
要使用IPython，请在运行bin / pyspark时将PYSPARK_DRIVER_PYTHON变量设置为ipython：
```python
$ PYSPARK_DRIVER_PYTHON=ipython ./bin/pyspark
```

使用Jupyter notebook：
```python
$ PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS=notebook ./bin/pyspark
```
您可以通过设置PYSPARK_DRIVER_PYTHON_OPTS来定制ipython或jupyter命令。

启动Jupyter Notebook服务器后，您可以从“文件”选项卡创建一个新的“ Python 2”笔记本。 
在笔记本内部，您可以在笔记本中开始内联输入命令％pylab作为笔记本的一部分，然后再从Jupyter笔记本开始尝试Spark。

# Resilient Distributed Datasets(RDDs)

Spark围绕弹性分布式数据集（RDD）的概念展开，RDD是可并行操作的元素的容错集合。 创建RDD的方法有两种：并行化驱动程序中的现有集合，或引用外部存储系统（例如共享文件系统，HDFS，HBase或提供Hadoop InputFormat的任何数据源）中的数据集。

## Parallelized Collections

通过在驱动程序中现有的可迭代对象或集合上调用SparkContext的parallelize方法来创建并行集合。 复制集合的元素以形成可以并行操作的分布式数据集。 例如，以下是创建包含数字1到5的并行化集合的方法：

```python
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
```
一旦创建了分布式数据集(distData)，就可以并行地操作它。例如，我们可以调用distData.reduce(lambda a, b: a + b)将列表中的元素相加。稍后我们将描述对分布式数据集的操作。

并行集合的一个重要参数是将数据集分割成的分区的数量。Spark将为集群的每个分区运行一个任务。通常，您希望集群中的每个CPU有2-4个分区。
通常，Spark尝试根据您的集群自动设置分区数量。但是，您也可以通过将它作为第二个参数传递来手动设置它(例如，sc.parallelize(data, 10))。注意:代码中的一些地方使用术语片(分区的同义词)来维护向后兼容

## External Datasets

PySpark可以从Hadoop支持的任何存储源创建分布式数据集，包括您的本地文件系统，HDFS，Cassandra，HBase，Amazon S3等。Spark支持文本文件，SequenceFiles和任何其他Hadoop InputFormat。

可以使用SparkContext的textFile方法创建文本文件RDD。 此方法获取文件的URI（计算机上的本地路径，或hdfs：//，s3a：//等URI），并将其读取为行的集合。 这是一个示例调用
```python
>>> distFile = sc.textFile("data.txt")
```
一旦创建distFile，distFile就可以进行数据集运算。例如：我们可以使用map和reduce函数计算所有行的长度：distFile.map(lambda s:len(s)).reduce(lambda a, b:a + b).

Spark读取文件的一些注意事项：
- 如果在本地文件系统上使用路径，则还必须在工作节点上的相同路径上访问该文件。 将文件复制到所有工作服务器，或者使用网络安装的共享文件系统。
- 所有基于文件的输入方法，包括textFile，支持在目录、压缩文件和通配符上运行。例如，可以使用textFile("/my/directory")、textFile("/my/directory/*.txt")和textFile("/my/directory/*.gz")。
- textFile方法还接受一个可选的第二个参数来控制文件的分区数量。默认情况下，Spark为文件的每个块创建一个分区(在HDFS中，块的默认大小为128MB)，但是您也可以通过传递更大的值来请求更多的分区。注意，分区不能少于块。

除文本文件外，Spark的Python API还支持其他几种数据格式：
- SparkContext.wholeTextFiles使您可以读取包含多个小文本文件的目录，并将每个小文本文件作为（文件名，内容）对返回。 这与textFile相反，后者将在每个文件的每一行返回一条记录。
- RDD.saveAsPickleFile和SparkContext.pickleFile支持以包含pickled的Python对象的简单格式保存RDD。 批处理用于pickle序列化，默认批处理大小为10。
- SequenceFile和Hadoop输入/输出格式

**Note** 此功能当前标记为“实验性”，仅供高级用户使用。 将来可能会替换为基于Spark SQL的读/写支持，在这种情况下，Spark SQL是首选方法。

**Writable Support**

PySpark SequenceFile支持在Java中加载键-值对的RDD，将可写对象转换为基本Java类型，并使用Pyrolite pickle生成的Java对象。将键/值对的RDD保存到SequenceFile时，PySpark会执行相反的操作。 它将Python对象分解为Java对象，然后将它们转换为Writables。 以下可写对象将自动转换：

Writable Type | Python Type
------ | ------
Text | unicode str
IntWritable | int
FloatWritable | float
DoubleWritable | float
BooleanWritable | bool
BytesWritable | bytearray
NullWritable | None
MapWritable | dict

数组不是开箱即用的。用户在读写时需要指定定制的ArrayWritable子类型。在编写时，用户还需要指定将数组转换为自定义ArrayWritable子类型的自定义转换器。读取时，默认转换器将自定义ArrayWritable子类型转换为Java Object[]，然后将其pickle为Python元组。获取Python数组。对于基元类型的数组，用户需要指定自定义转换器。

**Saving and Loading Other Hadoop Input/Output Formats**

对于“新的”和“旧的”Hadoop MapReduce api, PySpark可以读取任何Hadoop InputFormat或编写任何Hadoop OutputFormat。如果需要，可以将Hadoop配置作为Python dict传入。下面是一个使用Elasticsearch ESInputFormat的例子
```
$ ./bin/pyspark --jars /path/to/elasticsearch-hadoop.jar
>>> conf = {"es.resource" : "index/type"}  # assume Elasticsearch is running on localhost defaults
>>> rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",
                             "org.apache.hadoop.io.NullWritable",
                             "org.elasticsearch.hadoop.mr.LinkedMapWritable",
                             conf=conf)
>>> rdd.first()  # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
```

注意，如果InputFormat仅仅依赖于Hadoop配置和/或输入路径，并且键和值类可以根据上表轻松地进行转换，那么这种方法应该可以很好地用于这种情况。

如果您有自定义的序列化二进制数据(例如从Cassandra / HBase加载数据)，那么首先需要在Scala/Java端将数据转换为可以由Pyrolite的pickler处理的数据。为此提供了一个转换器特性。只需扩展此特征并在convert方法中实现您的转换代码。请记住，确保这个类以及访问InputFormat所需的任何依赖项都打包到Spark job jar中，并包含在PySpark类路径中。

有关使用定制转换器使用Cassandra / HBase InputFormat和OutputFormat的示例，请参阅[Python示例](https://github.com/apache/spark/tree/master/examples/src/main/python)和[转换器示例](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/pythonconverters)。

## RDD Operations

RDD支持两种类型的操作：转换（从现有操作创建新数据集）和动作（在数据集上运行计算后将值返回到驱动程序）。例如，map是一个转换操作，对数据集上的每一个元素进行函数操作，然后返回一个新的表示结果的RDD数据集另一方面，reduce是使用函数聚合RDD所有元素的操作，并返回最后的结果给驱动程序（尽管还有一个返回分布式数据集的reduceByKey）。

Spark中的所有转换操作都是惰性的，他们不会立即计算，而只是记住有哪些转换操作作用于相同的基础数据集。当需要返回结果给驱动器的时候，这些转换操作才会计算相应的结果。这种设计能够使Spark运行效率更高。例如，我们可以认识到通过map创建的数据集将用于reduce中，并且仅将reduce的结果返回给驱动程序，而不是将较大的maped数据集返回给驱动程序。

默认情况下，每次对转换后的RDD执行一个操作时，都需要重新计算它。然而，你可以通过persist(or cache)方法持久化一个RDD的内存中，在这种情况下，Spark会将元素保留在集群中，以便在下一次查询时更快地进行访问。还支持在磁盘上持久存储RDDs，或跨多个节点复制RDDs。

### Basics

为了说明RDD的基本原理，看下面这个例子：
```
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = linLengths.reduce(lambda a, b: a + b)
```
第一行从一个外部文件定义了一个基本的RDD。这个数据没有加载到内存，也没有其他任何作用，lines只是一个指向文件的指针。第二行定义了一个`lineLengths`来表示map转换操作的结果。由于延迟加载，lineLengths没有立即计算。最后我们运行reduce这个操作。此时，Spark将计算分解为任务运行在不同的机器上，并且每台机器都运行部分map操作和reduce操作，并且指向驱动程序返回结果。

如果我们也想再次使用lineLengths，我们可以在reduce前增加如下操作：
```
lineLengths.persist()
```
这将导致lineLengths在第一次计算后保存在内存中

### Passing Functions to Spark

Spark的API在很大程度上依赖于在驱动程序中传递函数以在群集上运行。 建议使用三种方法来执行此操作
- Lambda表达式，用于可以作为表达式编写的简单函数。 （Lambda不支持多语句函数或不返回值的语句。
- Spark中调用本地的函数。
- 模块中的顶级函数。

例如，要传递比lambda支持的函数更长的函数，请考虑以下代码：
```
"""MyScript.py"""
if __name__="__main__":
    def myFUnc(s):
        words = s.split(" ")
        return len(words)
    
    sc = SparkContext(...)
    sc.textFile("file.txt").map(myFunc)
```
请注意，虽然也可以将引用传递给类实例中的方法(与单例对象相反)，但这需要同时发送包含该类的对象和方法。举个例子：
```
class MyClass(Object):
    def func(self, s):
        return s
    def doStuff(self, rdd):
        return rdd.map(self.func)
```
在这里，如果我们创建一个新的MyClass并在其上调用doStuff，则其中的映射将引用该MyClass实例的func方法，因此需要将整个对象发送到集群。

以类似的方式，访问外部对象的字段将引用整个对象：
```
class MyClass(object):
    def __int___(self):
        self.fidle = "hello"
        
    def doStudff(self, rdd):
        return rdd.map(lambad s: self.field + s)
```
为了避免这个问题，最简单的方法是将字段复制到局部变量中，而不是从外部访问它:

```
def doStuff(self, rdd):
    field = self.field
    return rdd.map(lambda s: field + s)
```

### Understanding closures

Spark的难点之一是理解跨集群执行代码时变量和方法的范围和生命周期。修改超出其范围的变量的RDD操作可能经常引起混乱。在下面的示例中，我们将查看使用foreach()递增计数器的代码，但是其他操作也可能出现类似的问题。

#### Example

考虑下面简单的RDD元素sum，它的行为可能会根据是否在相同的JVM中执行而有所不同。一个常见的例子是，在本地模式下运行Spark(—master = local[n])与在集群中部署Spark应用程序(例如，通过Spark -submit to YARN)：
```
counter = 0
rdd = sc.parallelize(data)

# Wrong: Don't do this!!
def increment_counter(x):
    global counter
    counter += x
rdd.foreach(increment_counter)

print("Counter value: ", counter)
```

#### Local vs. cluster modes

上述代码的行为是未定义的，可能无法按预期工作。为了执行作业，Spark将RDD操作的处理分解为任务，每个任务由执行程序(task)执行。在执行之前，Spark计算任务的闭包。闭包是那些执行程序在RDD上执行其计算时必须可见的变量和方法(在本例中为foreach())。这个闭包被序列化并发送给每个执行器。

发送给每个执行者的闭包中的变量现在是副本，因此，在foreach函数中引用计数器时，它不再是驱动程序节点上的计数器。 驱动程序节点的内存中仍然存在一个计数器，但是执行者将不再看到该计数器！ 执行者仅从序列化闭包中看到副本。 因此，由于对计数器的所有操作都引用了序列化闭包内的值，所以计数器的最终值仍将为零。

在本地模式下，在某些情况下，foreach函数实际上将在与驱动程序相同的JVM中执行，并且将引用相同的原始计数器，并且可能会对其进行实际更新。

为了确保在此类情况下行为明确，应使用累加器。 Spark中的累加器专门用于提供一种机制，用于在集群中的各个工作节点之间拆分执行时安全地更新变量。 本指南的“累加器”部分将详细讨论这些内容。

通常，闭包-像循环或局部定义的方法之类的结构，不应用于突变某些全局状态。 Spark不定义或保证从闭包外部引用的对象的突变行为。 某些执行此操作的代码可能会在本地模式下工作，但这只是偶然的情况，此类代码在分布式模式下将无法正常运行。 如果需要一些全局聚合，请使用累加器。

##### Printing elements of an RDD

另一个常见用法是尝试使用rdd.foreach（println）或rdd.map（println）打印出RDD的元素。 在单台机器上，这将生成预期的输出并打印所有RDD的元素。 但是，在集群模式下，执行者正在调用stdout的输出现在写入执行者的stdout，而不是驱动程序上的那个，因此驱动程序上的stdout不会显示这些信息！ 要在驱动程序上打印所有元素，可以使用collect（）方法首先将RDD带到驱动程序节点：rdd.collect（）。foreach（println）。 但是，这可能会导致驱动程序用尽内存，因为collect（）将整个RDD提取到一台计算机上。 如果只需要打印RDD的一些元素，则更安全的方法是使用take（）：rdd.take（100）.foreach（println）。

### Working with Key-Value Pairs

尽管大多数Spark操作可在包含任何类型的对象的RDD上运行，但一些特殊操作仅可用于键-值对的RDD。 最常见的是分布式“混洗”操作，例如通过键对元素进行分组或聚合。

在Python中，这些操作在包含内置Python元组(1,2)的RDDs上工作.

例如，下面的代码在一个key-value对上执行reduceByKey来计算一个文件中每一行出现的次数：
```
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambad a, b: a + b)
```
例如，我们还可以使用counts.sortByKey（）对字母对进行排序，最后使用counts.collect（）将它们作为对象列表带回到驱动程序中。


### Transformations

下表列出了Spark常见的转换操作。详细信息请参考RDD API文档([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD)、[Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html)、[Python](https://spark.apache.org/docs/latest/api/python/pyspark.html#pyspark.RDD)、[R](https://spark.apache.org/docs/latest/api/R/index.html))和RDD函数对文档([Scala](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)、[Java](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaPairRDD.html))。

Transformation | Meaning
------ | ------
map(func) | 返回一个新的分布式数据集，该数据集是通过将源的每个元素传递给函数func形成的
filter(func) | -
flatMap(func) | 与map相似，但是每个输入项都可以映射到0个或多个输出项（因此func应该返回Seq而不是单个项）
mapPartitions(func) | 与map类似，但是在RDD的每个分区(块)上单独运行，所以func在类型为T的RDD上运行时必须是类型Iterator\<T\> => Iterator\<U\>。
mapPartitionsWithIndex(func) | -
sample(withReplacement, fraction, seed) | -
union(otherDataset) | -
intersection(otherDataset) | -
distinct([numPartitionos]) | -
groupByKey([numPartitions]) | -
reduceByKey(func, [numPartitions])	 | -
aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])	 | -
sortByKey([ascending], [numPartitions])	 | -
join(otherDataset, [numPartitions])	 | -
cogroup(otherDataset, [numPartitions])	 | -
cartesian(otherDataset)	 | -
pipe(command, [envVars])	 | -
coalesce(numPartitions)	 | -
repartition(numPartitions)	 | -
repartitionAndSortWithinPartitions(partitioner)	 | -


### Actions

Action | Meaning
------ | ------
reduce(func) | -
collect() | -
count() | -
first() | -
take(n) | -
takeSample(withReplacement, num, [seed]) | -
takeOrdered(n, [ordering]) | -
saveAsTextFile(path) | -
saveAsSequenceFile(path) (java and scala) | -
saveAsObjectFile(path) (java and scala) | -
countBykey() | -
foreach(func) | -

Spark RDD API还公开了某些操作的异步版本，例如foreachAsync for foreach，该版本立即将FutureAction返回给调用方，而不是在操作完成时阻止。 这可用于管理或等待动作的异步执行。

### Shuffle operations

Spark中的某些操作会触发一个称为shuffle的事件。 shuffle是Spark的一种用于重新分配数据的机制，因此可以跨分区对数据进行不同的分组。 这通常涉及跨执行程序和机器复制数据，从而使shuffle成为复杂且昂贵的操作。



