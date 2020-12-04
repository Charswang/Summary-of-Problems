# TaskSchedule源码阅读笔记

spark的Task的调度，我们要明白其调度过程，**其根据不同的资源管理器拥有不同的调度策略，因此也拥有不同的调度守护进程，这个守护进程管理着集群的资源信息，**spark提供了一个基本的守护进程的类，来完成与driver和executor的交互：<font color = 'red'>**CoarseGrainedSchedulerBackend**</font>，它应该运行在集群资源管理器上，比如yarn等。他收集了集群work机器的一般资源信息。当我们形成tasks将要进行调度的时候，driver进程会与其通信，请求资源的分配和调度，其会把最优的work节点分配给task来执行其任务。而<font color = 'red'>**TaskScheduleImpl实现了task调度的过程**</font>，采用的调度算法默认的是FIFO的策略，也可以采用公平调度策略。

![img](https://s4.51cto.com/images/blog/201803/26/fe4689198ce8ab54357c2cd4af897a98.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

当我们提交task时，其会创建一个管理task的类TaskSetManager，然后把其加入到任务调度池中。

----------------------------------------------

override def <font color = 'red'>**submitTasks(taskSet: TaskSet)**</font> {
val tasks = taskSet.tasks
<font color = 'red'>**logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")**</font>
this.synchronized {
// 创建taskSetManager，以下为更新一下状态
val manager = <font color = 'red'>**createTaskSetManager(taskSet, maxTaskFailures)**</font>
val stage = taskSet.stageId
val stageTaskSets =
taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
stageTaskSets(taskSet.stageAttemptId) = manager
val conflictingTaskSet = stageTaskSets.exists { case (*, ts) =>ts.taskSet != taskSet && !ts.isZombie}if (conflictingTaskSet) {throw new IllegalStateException(s"more than one active taskSet for stage $stage:" +s" ${stageTaskSets.toSeq.map{*._2.taskSet.id}.mkString(",")}")
}
//把封装好的taskSet，加入到任务调度队列中。
<font color = 'red'>**schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)**</font>

```
if (!isLocal && !hasReceivedTask) {
  starvationTimer.scheduleAtFixedRate(new TimerTask() {
    override def run() {
      if (!hasLaunchedTask) {
        logWarning("Initial job has not accepted any resources; " +
          "check your cluster UI to ensure that workers are registered " +
          "and have sufficient resources")
      } else {
        this.cancel()
      }
    }
  }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
}
hasReceivedTask = true
```

}
<font color = 'red'>**//这个地方就是向资源管理器发出请求，请求任务的调度**</font>
<font color = 'red'>**backend.reviveOffers()**</font>
}

/*
*这个方法是位于CoarseGrainedSchedulerBackend类中，driver进程会想集群管理器发送请求资源的请求。
/
override def reviveOffers() {
driverEndpoint.send(ReviveOffers)
}

----------------------------------------------

当其收到这个请求时，其会调用这样的方法。

----------------------------------------------

override def receive: PartialFunction[Any, Unit] = {
case StatusUpdate(executorId, taskId, state, data) =>
scheduler.statusUpdate(taskId, state, data.value)
if (TaskState.isFinished(state)) {
executorDataMap.get(executorId) match {
case Some(execxutorInfo) =>
executorInfo.freeCores += scheduler.CPUS_PER_TASK
makeOffers(executorId)
case None =>
// Ignoring the update since we don't know about the executor.
logWarning(s"Ignored task status update ($taskId state $state) " +
s"from unknown executor with ID $executorId")
}
}
//发送的请求满足这个条件
<font color = 'red'>**case ReviveOffers**</font> =>
makeOffers()

<font color = 'red'>**case KillTask(taskId, executorId, interruptThread)**</font> =>
executorDataMap.get(executorId) match {
<font color = 'red'>**case Some(executorInfo)**</font> =>
executorInfo.executorEndpoint.send(KillTask(taskId, executorId, interruptThread))
case None =>
// Ignoring the task kill since the executor is not registered.
logWarning(s"Attempted to kill task $taskId for unknown executor $executorId.")
}
}

/*
*<font color = 'red'>**这个方法是搜集集群上现在还在活着的机器的相关信息。并且进行封装成WorkerOffer类**</font>，

- <font color = 'red'>**然后其会调用TaskSchedulerImpl中的resourceOffers方法，来进行筛选，筛选出符合请求资源的机器，来执行我们当前的任务**</font>
  /
  private def <font color = 'red'>**makeOffers()**</font> {
  // Filter out executors under killing
  val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
  val workOffers = activeExecutors.map { case (id, executorData) =>
  new WorkerOffer(id, executorData.executorHost, executorData.freeCores)
  }.toIndexedSeq
  launchTasks(scheduler.resourceOffers(workOffers))
  }

<font color = 'red'>**/*得到集群中空闲机器的信息后，我们通过此方法来筛选出满足我们这次任务要求的机器，然后返回TaskDescription类**</font>
*<font color = 'red'>**这个类封装了task与excutor的相关信息**</font>

- /
  def resourceOffers(offers: IndexedSeq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
  // Mark each slave as alive and remember its hostname
  // Also track if new executor is added
  var newExecAvail = false
  <font color = 'red'>//检查work是否已经存在了，把不存在的加入到work调度池中</font>
  for (o <- offers) {
  if (!hostToExecutors.contains(o.host)) {
  hostToExecutors(o.host) = new HashSet[String]()
  }
  if (!executorIdToRunningTaskIds.contains(o.executorId)) {
  hostToExecutors(o.host) += o.executorId
  executorAdded(o.executorId, o.host)
  executorIdToHost(o.executorId) = o.host
  executorIdToRunningTaskIds(o.executorId) = HashSet[Long]()
  newExecAvail = true
  }
  for (rack <- getRackForHost(o.host)) {
  hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
  }
  }
  <font color = 'red'>// 打乱work机器的顺序，以免每次分配任务时都在同一个机器上进行。避免某一个work计算压力太大。</font>
  val shuffledOffers = Random.shuffle(offers)
  <font color = 'red'>//对于每一work，创建一个与其核数大小相同的数组，数组的大小决定了这台work上可以并行执行task的数目.</font>
  val **tasks** = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](https://blog.51cto.com/9269309/o.cores))
  <font color = 'red'>//取出每台机器的cpu核数</font>
  val availableCpus = shuffledOffers.map(o => o.cores).toArray
  <font color = 'red'>//从task任务调度池中，按照我们的调度算法，取出需要执行的任务</font>
  val sortedTaskSets = rootPool.getSortedTaskSetQueue
  for (taskSet <- sortedTaskSets) {
  logDebug("parentName: %s, name: %s, runningTasks: %s".format(
  taskSet.parent.name, taskSet.name, taskSet.runningTasks))
  if (newExecAvail) {
  **taskSet.executorAdded()**
  }
  }
  <font color = 'red'>// 下面的这个循环，是用来标记task根据work的信息来标定数据本地化的程度的。当我们在yarn资源管理器，以--driver-mode配置</font>
  <font color = 'red'>//为client时，我们就会在打出来的日志上看出每一台机器上运行task的数据本地化程度。同时还会选择每个task对应的work机器</font>
  // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
  for (taskSet <- sortedTaskSets) {
  var launchedAnyTask = false
  var launchedTaskAtCurrentMaxLocality = false
  for (currentMaxLocality <- taskSet.myLocalityLevels) {
  do {
  launchedTaskAtCurrentMaxLocality = resourceOfferSingleTaskSet(
  taskSet, currentMaxLocality, shuffledOffers, availableCpus, tasks)
  launchedAnyTask |= launchedTaskAtCurrentMaxLocality
  } while (launchedTaskAtCurrentMaxLocality)
  }
  if (!launchedAnyTask) {
  taskSet.abortIfCompletelyBlacklisted(hostToExecutors)
  }
  }

  if (tasks.size > 0) {
  hasLaunchedTask = true
  }
  //返回taskDescription对象
  return tasks
  }

<font color = 'red'>/*task选择执行其任务的work其实是在这个函数中实现的，从这个可以看出，一台work上其实是可以运行多个task，主要是看如何</font>
<font color = 'red'>进行算法调度</font>

- /
  private <font color = 'red'>**resourceOfferSingleTaskSet(**</font>
  taskSet: TaskSetManager,
  maxLocality: TaskLocality,
  shuffledOffers: Seq[WorkerOffer],
  availableCpus: Array[Int],
  tasks: IndexedSeq[ArrayBuffer[TaskDescription]]) : Boolean = {
  var launchedTask = false
  <font color = 'red'>//循环所有的机器，找适合此机器的task</font>
  for (i <- 0 until shuffledOffers.size) {
  val execId = shuffledOffers(i).executorId
  val host = shuffledOffers(i).host
  <font color = 'red'>//判断其剩余的cpu核数是否满足我们的最低配置，满足则为其分配任务，否则不为其分配任务。</font>
  if (availableCpus(i) >= CPUS_PER_TASK) {
  try {
  <font color = 'red'>//这个for中的resourOffer就是来判断其标记任务数据本地化的程度的。task(i)其实是一个数组，数组大小和其cpu核心数大小相同。</font>
  for (task <- taskSet.resourceOffer(execId, host, maxLocality)) {
  tasks(i) += task
  val tid = task.taskId
  taskIdToTaskSetManager(tid) = taskSet
  taskIdToExecutorId(tid) = execId
  executorIdToRunningTaskIds(execId).add(tid)
  availableCpus(i) -= CPUS_PER_TASK
  assert(availableCpus(i) >= 0)
  launchedTask = true
  }
  } catch {
  case e: TaskNotSerializableException =>
  logError(s"Resource offer failed, task set ${taskSet.name} was not serializable")
  // Do not offer resources for this task, but don't throw an error to allow other
  // task sets to be submitted.
  return launchedTask
  }
  }
  }
  return launchedTask
  }

  # ----------------------------------------------

  以上完成了从TaskSet到task和work机器的绑定过程的所有任务。下面就是如何发送task到executor进行执行。在makeOffers()方法中调用了launchTasks方法,这个方法其实就是发送task作业到指定的机器上。只此，spark TaskSchedule的调度就此结束。

![spark DAGScheduler、TaskSchedule、Executor执行task源码分析](https://s4.51cto.com/images/blog/201803/26/748ac8493b22d8c021d622a6e401071a.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)