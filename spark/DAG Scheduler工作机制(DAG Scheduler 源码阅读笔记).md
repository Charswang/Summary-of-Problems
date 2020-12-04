# DAG Scheduler 源码阅读笔记

spark的调度分为两级调度：DAGSchedule和TaskSchedule。DAGSchedule是根据job来生成相互依赖的stages，然后把stages以TaskSet形式传递给TaskSchedule来进行任务的分发过程

RDD的DAG的形成过程，是通过依赖来完成的，每一个RDD通过转换算子的时候都会生成一个和多个子RDD，在通过转换算子的时候，在创建一个新的RDD的时候，也会创建他们之间的依赖关系。因此他们是通过Dependencies连接起来的，RDD的依赖不是我们的重点，如果想了解RDD的依赖，可以自行google。

**RDD的依赖分为：1:1的OneToOneDependency，m:1的RangeDependency，还有m:n的ShuffleDependencies，其中OneToOneDependency和RangeDependency又被称为NarrowDependency，这里的1:1,m:1,m:n的粒度是对于RDD的分区而言的。**

spark的Application划分job其实挺简单的，一个Application划分为几个job，我们就要看这个Application中有多少个Action算子，一个Action算子对应一个job，这个可以通过源码来看出来，转换算子是形成一个或者多个RDD，而Action算子是触发job的提交

![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/9393282a321643a4aad0cf26bdd6f7cb.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
而Action算子是这样的：
![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/4c75ed8f0b3d2182afeca21829bf846b.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
通过runJob方法提交作业。stage的划分是根据是否进行shuflle过程来决定的，这个后面会细说。

**spark的DAGScheduler的调度**

当我们通过客户端，向spark集群提交作业时，如果利用的资源管理器是yarn，那么客户端向spark提交申请运行driver进程的机器，driver其实在spark中是没有具体的类的，**driver机器主要是用来运行用户编写的代码的地方，完成DAGScheduler和TaskScheduler，追踪task运行的状态。**记住，**用户编写的主函数是在driver中运行的，但是RDD转换和执行是在不同的机器上完成。**其实driver主要负责作业的调度和分发。Action算子到stage的划分和DAGScheduler的完成过程。

当我们在driver进程中运行用户定义的main函数的时候，首先会创建SparkContext对象，这个是我们与spark集群进行交互的入口它会初始化很多运行需要的环境，最主要的是初始化了DAGScheduler和TaskSchedule。

![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/c46f45b12aa0a3845e1d56f29e5cef3e.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)



我们以这样的的一个RDD的逻辑执行图来分析整个DAGScheduler的过程。

![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/02fafb1b87a9c8c2e32a63bfe6728ffe.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

因为DAGScheduler发生在driver进程中，我们就冲Driver进程运行用户定义的main函数开始。在上图中RDD9是最后一个RDD并且其调用了Action算子，就会触发作业的提交，其会调用SparkContext的runjob函数，其经过一系列的runJob的封装，会调用DAGScheduler的runJob

**在SparkContext中存在着runJob方法，但该runjob方法调用的是dagScheduler.runjob**

rdd: RDD[T], // rdd为上面提到的RDD逻辑执行图中的RDD9
func: (TaskContext, Iterator[T]) => U,这个方法也是RDD9调用Action算子传入的函数

**DAGScheduler的runJob,其又调用了submitJob函数**

//在这里会生成一个job的守护进程waiter，用来等待作业提交执行是否完成，其又调用了submitJob，其以下的代码都是用来处运行结果的一些log日志信息

**submitJob的源代码**

// 检查RDD的分区是否合法
//这一块是把我们的job继续进行封装到JobSubmitted，然后放入到一个进程中池里，spark会启动一个线程来处理我们提交的作业

在DAGScheduler类中有一个DAGSchedulerEventProcessLoop的类，用来接收处理DAGScheduler的消息事件

![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/c70638350981f60b5ba783819ab80369.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

**JobSubmitted对象，因此会执行第一个操作handleJobSubmitted，**在这里我们要说一下，Stage的类型，在spark中有两种类型的stage一种是ShuffleMapStage，和ResultStage，最后一个RDD对应的Stage是ResultStage，遇到Shuffle过程的RDD被称为ShuffleMapStage。

**dagScheduler.handleJobSubmitted函数**

//对应RDD9
// 先创建ResultStage(**利用createResultStage函数**)

//**创建ResultStage的时候，它会调用相关函数createResultStage**

private def <font color=red>**getOrCreateParentStages**</font>(rdd: RDD[_], firstJobId: Int): List[Stage] = {
<font color=red>**getShuffleDependencies**</font>(rdd).map { shuffleDep =>
<font color=red>**getOrCreateShuffleMapStage**</font>(shuffleDep, firstJobId)
}.toList
}
/**

**采用的是深度优先遍历找到Action算子的父依赖中的宽依赖。这个是最主要的方法，要看懂这个方法，其实后面的就好理解，最好结合这例子上面给出的RDD逻辑依赖图，比较容易看出来，根据上面的RDD逻辑依赖图，其返回的ShuffleDependency就是RDD2和RDD1，RDD7和RDD6的依赖,如果存在A<-B<-C,这两个都是shuffle依赖，那么对于C其只返回B的shuffle依赖，而不会返回A*/**
private[scheduler] def <font color=red>**getShuffleDependencies**</font>(
	rdd: RDD[*]): HashSet[ShuffleDependency[*, *,* ]] = {
	**//用来存放依赖**
	val parents = new HashSet[ShuffleDependency[*,* , *]]*



​	//遍历过的RDD放入这个里面val visited = new** HashSet[RDD[*]]



​	**//创建一个待遍历RDD的栈结构**
​	val waitingForVisit = new ArrayStack[RDD[*]]**//压入finalRDD，逻辑图中的RDD9***
​	waitingForVisit.push(rdd)**//循环遍历这个栈结构**
​	while (waitingForVisit.nonEmpty) {
​	val toVisit = waitingForVisit.pop()**// 如果RDD没有被遍历过执行其中的代码**
​	if (!visited(toVisit)) {
​	**//然后把其放入已经遍历队列中**
​		visited += toVisit**//得到依赖，我们知道依赖中存放的有父RDD的对象**
​		toVisit.dependencies.foreach {

​		**//如果这个依赖是shuffle依赖，则放入返回队列中**
​			case shuffleDep:ShuffleDependency[*, *,* ] =>
​				parents += shuffleDep
​			case dependency =>
​		**//如果不是shuffle依赖，把其父RDD压入待访问栈中，从而进行循环**
​				waitingForVisit.push(dependency.rdd)

​				}
​			}
​		}
​		parents

​	}

private def <font color=red>**getOrCreateShuffleMapStage**</font>(
    shuffleDep: ShuffleDependency[_, _, _],
    firstJobId: Int): ShuffleMapStage = {
    shuffleIdToMapStage.get(shuffleDep.shuffleId) match {
        case Some(stage) =>
        	stage
        case None =>

​        // Create stages for all missing ancestor shuffle dependencies.

​        <font color=red>**getMissingAncestorShuffleDependencies**</font>(shuffleDep.rdd).foreach { dep =>

​        // Even though getMissingAncestorShuffleDependencies only returns shuffle dependencies
​        // that were not already in shuffleIdToMapStage, it's possible that by the time we      
​        // get to a particular dependency in the foreach loop, it's been added to
​        // shuffleIdToMapStage by the stage creation process for an earlier dependency. See     
​        // SPARK-13902 for more information.

​        	if (!shuffleIdToMapStage.contains(dep.shuffleId)) {
​                <font color=red> **createShuffleMapStage**</font>(dep, firstJobId)
​            }
​        }

​        // Finally, create a stage for the given shuffle dependency.

​         <font color=red>**createShuffleMapStage**</font>(shuffleDep, firstJobId)
​    }
}

**/创建shuffleMapStage，根据上面得到的两个Shuffle对象，分别创建了两个shuffleMapStage**
//
def  <font color=red>**createShuffleMapStage**</font>(shuffleDep: ShuffleDependency[*,* , _], jobId: Int): ShuffleMapStage = {
	**//这个RDD其实就是RDD1和RDD6**
	val rdd = shuffleDep.rdd
	val numTasks = rdd.partitions.length
	val parents = getOrCreateParentStages(rdd, jobId) **//查看这两个ShuffleMapStage是否存在父Shuffle的Stage**
	val id = nextStageId.getAndIncrement()
	**//创建ShuffleMapStage，下面是更新一下SparkContext的状态**
	val stage = new ShuffleMapStage(
	id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep, mapOutputTracker)
	stageIdToStage(id) = stage
	shuffleIdToMapStage(shuffleDep.shuffleId) = stage
	updateJobIdStageIdMaps(jobId, stage)

​	if (!mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
​		// Kind of ugly: need to register RDDs with the cache and map output tracker here
​		// since we can't do it in the RDD constructor because # of partitions is unknown
​		logInfo("Registering RDD " + rdd.id + " (" + rdd.getCreationSite + ")")
​		mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length)
​	}
​	stage
}

**submitStage源代码比较简单，它会检查我们当前的stage依赖的父stage是否已经执行完成，如果没有执行完成会循环提交其父stage等待其父stage执行完成了，才提交我们当前的stage进行执行。**

/** Submits stage, but first recursively submits any missing parents. */

private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
        logDebug("submitStage(" + stage + ")")
        if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {    
            val missing = getMissingParentStages(stage).sortBy(_.id)
            logDebug("missing: " + missing)
            if (missing.isEmpty) {
                logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
                <font color=red>**submitMissingTasks**</font>(stage, jobId.get)
            } else {
                for (parent <- missing) {
                    submitStage(parent)
                }
                waitingStages += stage
            }
        }
    } else {
        abortStage(stage, "No active job for stage " + stage.id, None)
    }
}



private def <font color =red>**submitMissingTasks**</font>(stage: Stage, jobId: Int) {
logDebug("submitMissingTasks(" + stage + ")")
// Get our pending tasks and remember them in our pendingTasks entry
stage.pendingPartitions.clear()

```scala
// 计算需要计算的分区数
val partitionsToCompute: Seq[Int] = stage.findMissingPartitions()

// Use the scheduling pool, job group, description, etc. from an ActiveJob associated
// with this Stage
val properties = jobIdToActiveJob(jobId).properties

runningStages += stage

// 封装stage的一些信息，得到stage到分区数的映射关系，即一个stage对应多少个分区需要计算
stage match {
  case s: ShuffleMapStage =>
    outputCommitCoordinator.stageStart(stage = s.id, maxPartitionId = s.numPartitions - 1)
  case s: ResultStage =>
    outputCommitCoordinator.stageStart(
      stage = s.id, maxPartitionId = s.rdd.partitions.length - 1)
}
```

//得到每个分区对应的具体位置，即分区的数据位于集群的哪台机器上。
val taskIdToLocations: Map[Int, Seq[TaskLocation]] = try {
stage match {
case s: ShuffleMapStage =>
partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
case s: ResultStage =>
partitionsToCompute.map { id =>
val p = s.partitions(id)
(id, getPreferredLocs(stage.rdd, p))
}.toMap
}
} catch {
case NonFatal(e) =>
stage.makeNewStageAttempt(partitionsToCompute.size)
listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
runningStages -= stage
return
}
// 这个把上面stage要计算的分区和每个分区对应的物理位置进行了从新封装，放在了latestInfo里面
stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)
listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))

//序列化我们刚才得到的信息，以便在driver机器和work机器之间进行传输
var taskBinary: Broadcast[Array[Byte]] = null
try {
// For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
// For ResultTask, serialize and broadcast (rdd, func).
val taskBinaryBytes: Array[Byte] = stage match {
case stage: ShuffleMapStage =>
JavaUtils.bufferToArray(
closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef))
case stage: ResultStage =>
JavaUtils.bufferToArray(closureSerializer.serialize((stage.rdd, stage.func): AnyRef))
}

```scala
  taskBinary = sc.broadcast(taskBinaryBytes)
} catch {
  // In the case of a failure during serialization, abort the stage.
  case e: NotSerializableException =>
    abortStage(stage, "Task not serializable: " + e.toString, Some(e))
    runningStages -= stage

    // Abort execution
    return
  case NonFatal(e) =>
    abortStage(stage, s"Task serialization failed: $e\n${Utils.exceptionString(e)}", Some(e))
    runningStages -= stage
    return
}
```

//封装stage构成taskSet集合，ShuffleMapStage对应的task为ShuffleMapTask，而ResultStage对应的taskSet为ResultTask
val tasks: Seq[Task[_]] = try {
stage match {
case stage: ShuffleMapStage =>
partitionsToCompute.map { id =>
val locs = taskIdToLocations(id)
val part = stage.rdd.partitions(id)
new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
taskBinary, part, locs, stage.latestInfo.taskMetrics, properties, Option(jobId),
Option(sc.applicationId), sc.applicationAttemptId)
}

```scala
  case stage: ResultStage =>
    partitionsToCompute.map { id =>
      val p: Int = stage.partitions(id)
      val part = stage.rdd.partitions(p)
      val locs = taskIdToLocations(id)
      new ResultTask(stage.id, stage.latestInfo.attemptId,
        taskBinary, part, locs, id, properties, stage.latestInfo.taskMetrics,
        Option(jobId), Option(sc.applicationId), sc.applicationAttemptId)
    }
}
```

} catch {
case NonFatal(e) =>
abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
runningStages -= stage
return
}

//提交task给TaskSchedule
if (tasks.size > 0) {
logInfo("Submitting " + tasks.size + " missing tasks from " + stage + " (" + stage.rdd + ")")
stage.pendingPartitions ++= tasks.map(_.partitionId)
logDebug("New pending partitions: " + stage.pendingPartitions)
taskScheduler.submitTasks(new TaskSet(
tasks.toArray, stage.id, stage.latestInfo.attemptId, jobId, properties))
stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
} else {
// Because we posted SparkListenerStageSubmitted earlier, we should mark
// the stage as completed here in case there are no tasks to run
markStageAsFinished(stage, None)

```scala
val debugString = stage match {
  case stage: ShuffleMapStage =>
    s"Stage ${stage} is actually done; " +
      s"(available: ${stage.isAvailable}," +
      s"available outputs: ${stage.numAvailableOutputs}," +
      s"partitions: ${stage.numPartitions})"
  case stage : ResultStage =>
    s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})"
}
logDebug(debugString)

submitWaitingChildStages(stage)
```

}
}