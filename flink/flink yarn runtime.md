# Start a session yarn cluster
## yarn-session.sh
```
$ ./bin/yarn-session.sh -n 4 -jm 1024m -tm 1024m
```
解析出flink-dist和lib的路径，最后执行`FlinkYarnSessionCli`，带-j参数指定flink-dist的jar包
```
$JAVA_RUN $JVM_ARGS -classpath "$CC_CLASSPATH" $log_setting org.apache.flink.yarn.cli.FlinkYarnSessionCli -j "$FLINK_LIB_DIR"/flink-dist*.jar "$@"
```

## 启动session Cluster过程
1. startAppMaster
```
FlinkYarnSessionCli.main -> run
  createClusterDescriptor(cmd) // 创建YarnClusterDescriptor
  AbstractYarnClusterDescriptor.deploySessionCluster -> deployInternal
    yarnClient.createApplication()
  startAppMaster
    uploadAndRegisterFiles // 将资源上传HDFS
    setupSingleLocalResource // flink-dist jar包、flink-conf.yaml配置文件、job.graph单独处理
    setupApplicationMasterContainer // 设置appMaster容器配置，主要是启动命令，其中主入口在 YarnSessionClusterEntrypoint
    yarnClient.submitApplication(appContext); // 向yarn提交appMaster
  createYarnClusterClient // 创建RestClient
```

2. appMaster容器启动
从上一步的分析可以看到，appMaster容器实际上启动的是`YarnSessionClusterEntrypoint`
```
YarnSessionClusterEntrypoint.main
  ClusterEntrypoint.runClusterEntrypoint(yarnSessionClusterEntrypoint);
    ClusterEntrypoint.startCluster(); -> runCluster
      initializeServices(configuration); //初始化commonRpcService/haServices/blobServer/heartbeatServices等
      dispatcherResourceManagerComponentFactory = createDispatcherResourceManagerComponentFactory(configuration);
      dispatcherResourceManagerComponentFactory.create // DispatcherResourceManagerComponent包含了flink的各个组件
        LeaderRetrievalService dispatcherLeaderRetrievalService = null; 
		    LeaderRetrievalService resourceManagerRetrievalService = null; 
		    WebMonitorEndpoint<U> webMonitorEndpoint = null; 
		    ResourceManager<?> resourceManager = null; // 这里是YarnResourceManager
		    JobManagerMetricGroup jobManagerMetricGroup = null;
		    T dispatcher = null; // 负责接收job，产生一个JobManager去执行，这里是StandaloneDispatcher
```

# Submit a job
## flink
```
$ ./bin/flink run ./examples/batch/WordCount.jar --input hdfs:///user/ben/input/core-site.xml --output hdfs:///user/ben/output/result.txt
```
实际执行：
```
exec $JAVA_RUN $JVM_ARGS "${log_setting[@]}" -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" org.apache.flink.client.cli.CliFrontend "$@"
```
主入口在`CliFrontend`:
```
CliFrontend.main
  loadCustomCommandLines // 这里拿到实际的入口类 FlinkYarnSessionCli
  cli.parseParameters(args) // 解析参数
    run(params)
      buildProgram(runOptions) // 通过传过去的jar包和classpath构造一个PackagedProgram
      runProgram(customCommandLine, commandLine, runOptions, program); 
        customCommandLine.createClusterDescriptor(commandLine) // 创建一个ClusterDescriptor，这里的commandLine就是FlinkYarnSessionCli，创建了YarnClusterDescriptor
        customCommandLine.getClusterId(commandLine); // 获取cluster节点的id。在上面的yarn-session集群启动的时候会将节点id写入文件中，
                                                    // 这里去读取。对于per-job模式，这个id为空，根据是否是detach模式创建不同的集群
        client = clusterDescriptor.retrieve(clusterId); // 返回一个RestClusterClient
        executeProgram(program, client, userParallelism);
          client.run(program, parallelism); // 这里走的interactive模式，比较诡异
            ContextEnvironmentFactory factory = new ContextEnvironmentFactory(...)
            ContextEnvironment.setAsContext(factory); // 这里设置的factory在用户代码的ExecutionEnvironment.getExecutionEnvironment()用到
            prog.invokeInteractiveModeForExecution();
              callMainMethod(mainClass, args); // 这里实际执行用户代码的main方法入口，
                                               // 真正的提交工作其实是在ExecutionEnvironment.execute中进行
```

# Per-job & detach mode
## Per-job
上面第一种模式是启动一个session cluster，一直在监听，有任务提交就执行，没有的话就资源闲置。per-job这个模式下每次提交一个job都会单独创建一个session cluster，当job执行完成后，对应的session cluster会被关掉。
```
$ ./bin/flink run -m yarn-cluster -yjm 1024m -ytm 1024m ./examples/batch/WordCount.jar --input hdfs:///user/ben/input/core-site.xml --output hdfs:///user/ben/output/result.txt
```
主要区别在于`-m`参数指定了JobManager，在`CliFrontend.runProgram-> customCommandLine.getClusterId(commandLine);`得到的clusterId为空，要先去创建session 集群`client = clusterDescriptor.deploySessionCluster(clusterSpecification);`，且在执行完成后的finally中关闭集群`client.shutDownCluster();`。其他跟上面的session模式一样。

## detach mode
detach模式可以用于上面两种模式，只要增加`-d`参数即可。
```
./bin/flink run -d ./examples/batch/WordCount.jar --input hdfs:///user/ben/input/core-site.xml --output hdfs:///user/ben/output/result.txt

./bin/flink run -d -m yarn-cluster -yjm 1024m -ytm 1024m ./examples/batch/WordCount.jar --input hdfs:///user/ben/input/core-site.xml --output hdfs:///user/ben/output/result.txt
```
### one-session-multi-job
主要的区别在于`ClusterClient.run`如果是detach模式返回的类型不一样:`return ((DetachedEnvironment) factory.getLastEnvCreated()).finalizeExecute();`

### per-job
先生成JobGraph，然后自己创建cluster，不会去关闭集群，需要通过yarn去kill
```
final JobGraph jobGraph = PackagedProgramUtils.createJobGraph(program, configuration, parallelism);
final ClusterSpecification clusterSpecification = customCommandLine.getClusterSpecification(commandLine);
client = clusterDescriptor.deployJobCluster(
					clusterSpecification,
					jobGraph,
					runOptions.getDetachedMode());
```
`YarnClusterDescriptor.deployJobCluster`也是调用`deployInternal`，最大的区别是`yarnClusterEntrypoint`参数变成`YarnJobClusterEntrypoint`，且`jobGraph`不为空，在`startAppMaster`中会将其写到文件并加入到资源中，传给appMaster。在appMaster启动的时候调用`ClusterEntrypoint.runCluster->createDispatcherResourceManagerComponentFactory(configuration);`实际上调用的是`YarnJobClusterEntrypoint.createDispatcherResourceManagerComponentFactory`，初始化的`jobGraphRetriever`会读取上面的`jobGraph`，`dispatcherFactory.createDispatcher`的时候将`jobGraph`传递给`MiniDispatcher`。

因此这个和上面的per-job模式差别比较大，这里是创建集群的时候就已经把任务`jobGraph`传过去，直接执行；上面是先创建一个集群，再走正常的提交。


# 方法真正执行的入口
## Client端
```
batch:
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
  contextEnvironmentFactory.createExecutionEnvironment();
env.execute("WordCount Example") -> ContextEnvironment.execute(String jobName)
  Plan p = createProgramPlan(jobName);
	JobWithJars toRun = new JobWithJars(p, this.jarFilesToAttach, this.classpathsToAttach, this.userCodeClassLoader);
  client.run(toRun, getParallelism(), savepointSettings).getJobExecutionResult();
    OptimizedPlan optPlan = getOptimizedPlan(compiler, jobWithJars, parallelism);
    run(optPlan, jobWithJars.getJarFiles(), jobWithJars.getClasspaths(), classLoader, savepointSettings);
  

streaming:
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
  ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
  return new StreamContextEnvironment((ContextEnvironment) env);
env.execute("Streaming WordCount") -> StreamContextEnvironment.execute(String jobName)
  StreamGraph streamGraph = this.getStreamGraph();
  return ctx
				.getClient()
				.run(streamGraph, ctx.getJars(), ctx.getClasspaths(), ctx.getUserCodeClassLoader(), ctx.getSavepointRestoreSettings())
				.getJobExecutionResult();

```

这里`getExecutionEnvironment`都和上面`ClusterClient.run(PackagedProgram prog, int parallelism)`中的代码有关：
```
ContextEnvironmentFactory factory = new ContextEnvironmentFactory(this, libraries,
					prog.getClasspaths(), prog.getUserCodeClassLoader(), parallelism, isDetached(),
					prog.getSavepointSettings());
ContextEnvironment.setAsContext(factory);
```
即用到的都是通过`ContextEnvironmentFactory.createExecutionEnvironment`产生的`ContextEnvironment`，只是`StreamContextEnvironment`多加了一层封装。

batch和streaming最终走到同一个入口: 
```
ClusterClient.run(FlinkPlan compiledPlan, List<URL> libraries, List<URL> classpaths, ClassLoader classLoader, SavepointRestoreSettings savepointSettings)
  JobGraph job = getJobGraph(flinkConfig, compiledPlan, libraries, classpaths, savepointSettings);
	RestClusterClient.submitJob(job, classLoader) -> submitJob(jobGraph); // 这里实际上是RestClusterClient，将jobGraph提交给dispatcher
    jobGraphFileFuture // 将jobGraph写入文件
    requestFuture // 一个tuple，第一部分是请求的body，是第二部分的名称集合；第二部分是要上传的文件，包括上一步的jobGraph，还有这个任务的依赖
    submissionFuture // 提交请求
      sendRetriableRequest(...)
        restClient.sendRequest(webMonitorBaseUrl.getHost(), webMonitorBaseUrl.getPort(), messageHeaders, messageParameters, request, filesToUpload);
```

## Server端
上面Client端最终由restClient将请求发生出去，注意发送的地址是从`webMonitorBaseUrl`获取的，那Server端应该就是WebMonitor了。真正的实现类是`WebMonitorEndpoint`，在Yarn session下更具体点是`DispatcherRestEndpoint`:
```
ClusterEntrypoint.startCluster -> runCluster
  DispatcherResourceManagerComponentFactory<?> dispatcherResourceManagerComponentFactory = createDispatcherResourceManagerComponentFactory(configuration); 
  dispatcherResourceManagerComponentFactory.create(...) // 这里是SessionDispatcherResourceManagerComponentFactory
    webMonitorEndpoint = restEndpointFactory.createRestEndpoint(...) // restEndpointFactory是SessionDispatcherFactory
      return new DispatcherRestEndpoint(...)
```

`DispatcherRestEndpoint`(这个命名大概可以理解为包含`Dispatcher`的`WebMonitorEndpoint`，webMonitor本身没有提交job的功能，需要通过dispatcher实现)是一个netty-base的rest Server，除了继承`WebMonitorEndpoint`的Handler外还有自己特有的handler: `JobSubmitHandler`，刚好是用来接收client端上传的job请求。
```
JobSubmitHandler.handleRequest(...)
  CompletableFuture<JobGraph> jobGraphFuture = loadJobGraph(requestBody, nameToFile);
	Collection<Path> jarFiles = getJarFilesToUpload(requestBody.jarFileNames, nameToFile);
	Collection<Tuple2<String, Path>> artifacts = getArtifactFilesToUpload(requestBody.artifactFileNames, nameToFile);
  CompletableFuture<JobGraph> finalizedJobGraphFuture = uploadJobGraphFiles(gateway, jobGraphFuture, jarFiles, artifacts, configuration);
	CompletableFuture<Acknowledge> jobSubmissionFuture = finalizedJobGraphFuture.thenCompose(jobGraph -> gateway.submitJob(jobGraph, timeout));
```
`JobSubmitHandler`差不多对应client端的处理：
- 获取jobGraph
- 获取jar包
- 获取其他artifact
- 上传上面这些资源
- 通过gateway提交job

### Dispatcher
上面的gateway是`StandaloneDispatcher`，对应的submitJob在`Dispatcher`上实现:
```
Dispatcher.submitJob
  waitForTerminatingJobManager(jobId, jobGraph, this::persistAndRunJob) -> persistAndRunJob(JobGraph jobGraph) 
    runJob(jobGraph);
      createJobManagerRunner(jobGraph);
        jobManagerRunnerFactory.createJobManagerRunner(...) // 创建JobManagerRunner，负责job级别的leader election，包含一个JobMaster，负责执行一个job
          this.jobMaster = new JobMaster(...)
            this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup); // ExecutionGraph是协调dataflow执行的核心
        startJobManagerRunner(JobManagerRunner jobManagerRunner)
          jobManagerRunner.start(); 
            leaderElectionService.start(this); // 默认没有配置HA，使用StandaloneHaServices，产生的LeaderElectionService为StandaloneLeaderElectionService
              contender.grantLeadership(HighAvailabilityServices.DEFAULT_LEADER_ID); // 回调JobManagerRunner
                JobManagerRunner.verifyJobSchedulingStatusAndStartJobManager(leaderSessionID);
                  jobMaster.start(new JobMasterId(leaderSessionId), rpcTimeout); // jobMaster在这里启动
```

### JobMaster
Job真的是执行由`JobMaster`处理:
```
JobMaster.start(final JobMasterId newJobMasterId, final Time timeout)
  startJobExecution(newJobMasterId)
    startJobMasterServices();
      slotPool.start(getFencingToken(), getAddress());
      resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());
    resetAndScheduleExecutionGraph();
      scheduleExecutionGraph() // 有可能有多个job，需要调度，实际上是交给ExecutionGraph
        executionGraph.scheduleForExecution();
```

### ExecutionGraph
`scheduleForExecution`会按照`scheduleMode`进行调度，`ClusterClient.run(FlinkPlan compiledPlan, List<URL> libraries, List<URL> classpaths, ClassLoader classLoader, SavepointRestoreSettings savepointSettings)`这个入口里面创建了JobGraph，跟着下去可以发现batch使用的是`ScheduleMode.LAZY_FROM_SOURCES`，即可以等上游输入准备好了再开始下游，对应`scheduleLazy`方法；streaming使用`ScheduleMode.EAGER`，所有task必须立即调度，对应`scheduleEager(SlotProvider slotProvider, final Time timeout)`方法。方法里面都是会通过`SlotManager`开始申请资源。
