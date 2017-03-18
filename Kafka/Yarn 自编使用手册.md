# <center>Yarn 自编使用手册</center>
### 启动Spark模式
在Yarn上启动Spark有两种模式，在cluster模式下，Spark驱动器（driver）在Yarn Application Master中运行（运行于集群中），因此客户端可以在Spark应用程序启动之后关闭退出。而client模式，Spark驱动器在客户端进程中，Yarn Application Master只是向Yarn申请资源。

	./bin/spark-submit --class org.apache.spark.examples.SparkPi \
	    --master yarn \
	    --deploy-mode cluster \
	    --driver-memory 4g \
	    --executor-memory 2g \
	    --executor-cores 1 \
	    --queue thequeue \
	    lib/spark-examples*.jar 10

很明显，cluster模式适用于生产环境，而client模式适用于开发环境。

### cluster模式下添加Jar包
driver不在客户端机器上运行，SparkContext.addJar添加客户端本地文件就不行了，要使用客户端本地文件能够用SparkContext.addJar来添加，可以用--jars选项：

	./bin/spark-submit --class my.main.Class \
	    --master yarn \
	    --deploy-mode cluster \
	    --jars my-other-jar.jar,my-other-other-jar.jar my-main-jar.jar
    	app_arg1 app_arg2

### spark分配多少内存？
建议给Spark分配75%的内存，剩下的内存留给操作系统和系统缓存使用。

### Container
Container是RM（ResourceManager）分配资源的基本单位，每个Container包含特定数量的CPU和内存，Container有点像虚拟机。RM负责接收用户的资源并分配Container，NM（NodeManager）负责启动Container并监控资源使用。如果资源超出Container的限制，相应进程会被NM杀掉。

Container不仅用于资源分配，也用于资源隔离。

Container可以让客户端随意在指定的节点上分配，这样用户的程序可以只在特定的节点上执行。这样可以满足计算本地性的要求。

### 调度器和队列
在Yarn中，调度器是一个可插拔的组件，常见有FIFO，CapacitySheduler，FairScheduler的方式。

在RM端，根据不同的调度器，所有的资源被分为一个或多个队列（queue），每个队列包含一定量的资源，用户的每个application会被分配到一个队列中去执行，队列决定了用户能使用的资源上限。

所谓资源调度，就是决定将资源分配给哪个队列，哪个application的过程。

调度器的两个主要功能：
- 1.决定如何划分队列；
- 2.决定如何分配资源。

此外还有其他的特性：ACL，抢占，延迟调度等等。

### FIFO
最简单，默认的调度器。只有一个队列，所有用户共享。

先到先得的资源分配策略，但容易出现一个用户占满集群所有资源的情况。可以设置ACL，但不能设置各个用户的优先级。嘿嘿，这种模式一般不用于生产环境中。

### CapacityScheduler
在FIFO基础上，增加多用户支持，最大化集群吞吐量和利用率。简单理解是，集群空闲时，可以使用整个集群的资源，但是资源紧张时，也不能一个任务把资源全占了，每个用户都可以使用特定量的资源。基本提高整个集群的利用率，避免集群有资源但不能提交任务的情况。

博大精深的CapacityScheduler

- 划分队列使用xml文件配置，每个队列可以使用特定百分比的资源。
- 队列可以是树状结构，子队列资源之和必须100%，只有子队列能提交任务。
- 可以为每个队列设置ACL，哪些用户可以提交任务，哪些用户有admin权限。ACL可以继承。
- 队列资源可以动态变化，最多可以占用100%的资源，管理员也可以手动设置上限。
- 配置可以动态加载，只能添加队列，不能删除。
- 可以限制整个集群或每个队列的并发任务数量。
- 可以限定AM使用的资源比例，避免所有资源用来执行AM而只能无限期等待的情况。

CapacityScheduler调度时默认是只考虑内存的。

CapacityScheduler的问题就是某个用户的程序最多占用100%的资源，一直不主动释放，其他用户只能等待，因为其不支持抢占调度。

### FairScheduler
和CapacityScheduler不同，优先保证“公平”，每个用户只有特定数量的资源可以用，不能超出这个限制，即使集群整体很空闲。
特点：

- 使用xml文件配置，每个队列可以使用特定数量的内存和CPU。
- 队列是树状结构，只有叶子节点能提交任务。
- 可以为每个队列设置爱ACL
- 可以设置每个队列的权重
- 配置可以动态加载
- 可以限制集群，队列，用户的并发任务数量
- 支持抢占式调度

其不好的地方就是：有时候出现某个队列资源用满了，但是整体资源还比较闲的情况。整体的资源没有充分利用。





