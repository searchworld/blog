# AkkaRpcService
基于Akka+动态代理实现的RPC

## startServer
根据`RpcEndpoint`创建一个到`AkkaRpcActor`的`ActorRef`，然后封装一个`AkkaInvocationHandler/FencedAkkaInvocationHandler`，最终通过动态代理创建一个本地引用的`RpcServer`

## connect
根据`address`参数找到对应的`ActorRef`，然后也是封装一个`AkkaInvocationHandler`，通过动态代理返回一个`RpcGateway`，用于本地和远程交互

## AkkaInvocationHandler
上面的`startServer`和`connect`主要区别在于前者需要创建Actor，后者需要根据地址找到Actor，远程交互逻辑都是在`AkkaInvocationHandler`实现。

动态代理中的`InvocationHandler`，rpc实现逻辑的地方，主要函数`invokeRpc`:
```
// 根据要调用的方法名称和参数创建一个RpcInvocation
RpcInvocation rpcInvocation = createRpcInvocationMessage(methodName, parameterTypes, args);
  // 如果是本地调用，不需要序列化
  rpcInvocation = new LocalRpcInvocation(...)
  // 如果是远程调用，需要序列化
  RemoteRpcInvocation remoteRpcInvocation = new RemoteRpcInvocation(...)
// 如果返回参数为Void，则调用tell
tell(rpcInvocation);
// 如果返回参数不为空，则调用ask
CompletableFuture<?> futureResult = ask(rpcInvocation, futureTimeout);
```

这里的调用只是将方法名/参数封装成一个`RpcInvocation`，通过`startServer`和`connect`获取的`ActorRef`发送出去。

## AkkaRpcActor
这里接收到`RpcInvocation`，取出方法名和参数，真正发起方法调用。
```
handleRpcInvocation
  String methodName = rpcInvocation.getMethodName();
  Class<?>[] parameterTypes = rpcInvocation.getParameterTypes();
  rpcMethod = lookupRpcMethod(methodName, parameterTypes);
  result = rpcMethod.invoke(rpcEndpoint, rpcInvocation.getArgs());
```

## Fenced
`FencedRpcEndpoint/FencedAkkaRpcActor/FencedAkkaInvocationHandler`组成一组带fenced的rpc，和不带Fenced的区别是每个rpc消息都会携带`fencing tokens`，`rpc endpoint`自身也携带token，只有消息的token和endpoint的token一致的时候rpc才会执行。

代码上`FencedAkkaInvocationHandler`的tell和ask会先将消息封装成`FencedMessage`再发送出去，`FencedAkkaRpcActor.handleRpcMessage`会先对比`fencing tokens`，相同再发起调用，否则抛出异常。

`Dispatcher/JobMaster/ResourceManager`都是带`fenced`，使用的token实际上就是自身的id(和leader选举有关)，`SlotPool/TaskExecutor`不带`fenced`

# TaskExecutor
作为例子，`TaskExecutor`实现了`RpcEndpoint`，本身就是一个`RpcServer`；同时也实现了`RpcGateway`，可以作为远程调用的本地引用。

## startServer
在初始化的时候调用`RpcEndpoint`的初始化，`this.rpcServer = rpcService.startServer(this);`启动一个`RpcServer`

## connect
```
JobMaster.registerTaskManager
  AkkaRpcService.connect // 这里返回TaskExecutorGateway
    registeredTaskManagers.put(taskManagerId, Tuple2.of(taskManagerLocation, taskExecutorGateway));

JobMaster.offerSlots
  Tuple2<TaskManagerLocation, TaskExecutorGateway> taskManager = registeredTaskManagers.get(taskManagerId);
  final TaskManagerLocation taskManagerLocation = taskManager.f0;
  final TaskExecutorGateway taskExecutorGateway = taskManager.f1;
  RpcTaskManagerGateway rpcTaskManagerGateway = new RpcTaskManagerGateway(taskExecutorGateway, getFencingToken()); // 持有TaskExecutorGateway的引用
  slotPoolGateway.offerSlots(taskManagerLocation,rpcTaskManagerGateway,slots); // 进入SlotPool
    offerSlot(taskManagerLocation,taskManagerGateway,offer)
      AllocatedSlot allocatedSlot = new AllocatedSlot(...,taskManagerGateway); 

Execution.deploy
  TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();
  submitResultFuture = taskManagerGateway.submitTask(deployment, rpcTimeout); //RpcTaskManagerGateway
    taskExecutorGateway.submitTask(tdd, jobMasterId, timeout); // TaskExecutor
```
每个`Execution`都会分配到一个`LogicalSlot(assignedResource)`，这个slot里面持有`TaskManagerGateway`的引用，再持有`TaskExecutorGateway`，由`AkkaRpcService.connect`产生。