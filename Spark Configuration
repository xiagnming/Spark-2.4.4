# Spark Configuration
Spark提供了三个位置来配置系统：
- Spark属性控制大多数应用程序参数，可以使用SparkConf对象或Java系统属性来设置
- 环境变量可用于通过每个节点上的conf/spark-env.sh脚本设置每台机器的设置，如IP地址。
- 日志可以通过log4j.properties设置

## Spark Properties

Spark属性控制大多数应用程序设置，并分别为每个应用程序配置。可以直接在传递给SparkContext的SparkConf上设置这些属性。SparkConf允许您配置一些公共属性(例如主URL和应用程序名称)，以及通过set()方法配置任意键值对。例如，我们可以用以下两个线程初始化一个应用程序:

请注意，我们使用local [2]运行，这意味着两个线程-表示“最小”并行度，这可以帮助检测仅当我们在分布式上下文中运行时才存在的错误。
```
val conf = new SparkConf().setMaster("local[2]").setAppName("CountingSheep")
val sc = new SparkContext(conf)
```
请注意，在本地模式下我们可以有多个线程，并且在类似Spark Streaming的情况下，我们实际上可能需要多个线程来防止任何饥饿问题。

指定一定持续时间的属性应配置一个时间单位。 接受以下格式：
```
25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years)
```
指定字节大小的属性应配置为大小单位。 接受以下格式：
```
1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes)
```
通常将不带单位的数字解释为字节，而将少数解释为KiB或MiB。 请参阅各个配置属性的文档。 在可能的情况下，指定单位是可取的。

## Dynamically Loading Spark Properties

在某些情况下，您可能希望避免在SparkConf中对某些配置进行硬编码。例如，如果希望用不同的主机或不同的内存运行相同的应用程序。Spark允许您简单地创建一个空的conf
```
val sc = new SparkContext(new SparkConf())
```
然后，可以在运行时指定配置属性：
```
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```
Spark shell和spark-submit工具支持两种方式自动加载配置。第一种是命令行选项，像--master，spark-submit接受任何spark属性：使用--conf标识，但对在启动spark应用程序中起作用的属性使用特殊标志。运行./bin/spark-submit --help将显示这些选项的完整列表。

bin/spark-submit也可以从conf/spark-defaults.conf中读取配置，其中每一行由一个键和一个用空格分隔的值组成。例如
```
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
```
指定为标志或属性文件中的任何值都将传递到应用程序，并与通过SparkConf指定的那些值合并。 直接在SparkConf上设置的属性具有最高优先级，然后将标志传递到spark-submit或spark-shell，然后是spark-defaults.conf文件中的选项。自早期版本的Spark起，一些配置密钥已经重命名； 在这种情况下，较旧的密钥名称仍会被接受，但其优先级低于较新密钥的任何实例。

Spark属性主要可以分为两种：一种与部署相关，例如“ spark.driver.memory”，“ spark.executor.instances”，在运行时通过SparkConf进行编程设置时，此类属性可能不会受到影响；或者 该行为取决于您选择的集群管理器和部署模式，因此建议您通过配置文件或spark-submit命令行选项进行设置； 另一个主要与Spark运行时控件有关，例如“ spark.task.maxFailures”，可以用任何一种方式设置这种属性。

## Viewing Spark Properties

位于http：// <driver>：4040的应用程序Web UI在“环境”选项卡中列出了Spark属性。 这是检查以确保正确设置属性的有用位置。 请注意，只有通过spark-defaults.conf，SparkConf或命令行明确指定的值才会出现。 对于所有其他配置属性，您可以假定使用默认值。

## Available Properties

控制内部设置的大多数属性具有合理的默认值。 一些最常见的设置是：

### Application Properties

Property Name | default | Meaning
------ | ------ | -------
spark.app.name | none | 应用名字。将会出现在ui和日志数据中
spark.driver.cores | 1 | 驱动进程使用的内核数量，仅在集群模式下。



