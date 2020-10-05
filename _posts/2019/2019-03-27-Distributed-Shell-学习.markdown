---
layout: article
key: 734d8182-d65c-437d-b481-ed6fab05d8d0
title:  "Distributed Shell!"
date:   2019-03-27 15:29:12 +0800
categories: hadoop
tags: hadoop
---


* `ApplicationMaster`
* `Client`: 客户端，提交Application
* `DistributedShellTimelinePlugin`: Timeline v1.5 reader插件，作用未知？ 
* `DSConstants`: ApplicationMaster类与Client类中的一些常数
* `Log4jPropertyHelper`: 加载Log4j配置信息
* `PlacementSpec`: 一个封装SourceTag, container数目以及Placement约束的类。


#### 问题列表
* `Timeline`是做什么的？进而有一个`domain ID`怎么理解？
* 在类`ApplicationSubmissionContext`中的方法`setKeepContainersAcrossApplicationAttempts`，`For unmanaged AM, if the flag is true, RM allows re-register and returns the running containers in the same attempt back to the UAM(Unmanaged Application Master) for HA(?).`?
* 设置security tokens？其目的？
* 在ApplicationMaster中的RMCallbackHandler中的onContainersCompleted中，指定与不指定placementSpecs的区别是什么？
```java
           // Dont bother re-asking if we are using placementSpecs
           if (placementSpecs == null) {
                if (askCount > 0) {
                    for (int i = 0; i < askCount; ++i) {
                        ContainerRequest containerAsk = setupContainerAskForRM();
                        amRMClient.addContainerRequest(containerAsk);
                    }
                }
            }
```
* 在job完成后，ApplicationMaster~~发起FinishApplicaitionMasterRequest请求~~给RM，完成AM的注销。？ 源代码的语句在哪里？没看到这一请求的地方。
#### 已解决问题
* ApplicationsManager(ASM)，它是RM的一部分
* placement specification? 这是PlacementSpec类，表示sourceTag，container数目，placement约束的包装类。

[TOC]
#### `Client`类

##### 源码逻辑
`Client`类启动一个application master，AM将启动多个containers运行shell命令或者脚本。

* 连接到RM，创建一个新的Application。
通过YarnClient获取了集群信息、AM队列信息、访问控制列表信息、RM资源配置信息、验证 Application资源配置与container资源配置文件，同样适用YarnClient新建一个新的Application应用。

```java
YarnClientApplication app = yarnClient.createApplication();
GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
```
* 验证相关资源配置是否超出限制，现在仅有CPU和内存两种资源

* 在一个job提交过程中，Client首先创建一个ApplicationSubmissionContext，获取ApplicationId、设置application名字、设置applicationTags。
```java
ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
ApplicationId appId = appContext.getApplicationId();
```

* ApplicationSubmissionContext定义了Application的详细信息，例如：ApplicationId、ApplicationName、Application分配的优先级、Application分配的队列。

* 运行用户程序Container所需要的资源，AM并不需要（不必给AM Container提供，仅上传hdfs），但是启动其他的Container需要访问这些资源，因此AM需要知道这些资源的相关元数据（在hdfs上的路径、最后修改时间、文件长度）。
```java
        String hdfsShellScriptLocation = "";
        long hdfsShellScriptLen = 0;
        long hdfsShellScriptTimestamp = 0;
        if (!shellScriptPath.isEmpty()) {
            Path shellSrc = new Path(shellScriptPath);
            String shellPathSuffix =
                    ApplicationMaster.getRelativePath(appName,
                            appId.toString(),
                            SCRIPT_PATH);
            Path shellDst =
                    new Path(fs.getHomeDirectory(), shellPathSuffix);
            fs.copyFromLocalFile(false, true, shellSrc, shellDst);
            hdfsShellScriptLocation = shellDst.toUri().toString();
            FileStatus shellFileStatus = fs.getFileStatus(shellDst);
            hdfsShellScriptLen = shellFileStatus.getLen();
            hdfsShellScriptTimestamp = shellFileStatus.getModificationTime();
        }
```
作为AM Container的环境变量
```java
        Map<String, String> env = new HashMap<String, String>();
        // put location of shell script into env
        // using the env info, the application master will create the correct local resource for the
        // eventual containers that will be launched to execute the shell scripts
        env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
        env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTTIMESTAMP, Long.toString(hdfsShellScriptTimestamp));
        env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLEN, Long.toString(hdfsShellScriptLen));
```

* 在ApplicationSubmissionContext中包含一个ContainerLaunchContext对象，来描述运行AM自身的Container。在ContainerLaunchContext中，定义了AM的Container的资源需求：
    1) 运行AM的Container的资源：本地资源（JAR、配置文件）
    ```java
        FileSystem fs = FileSystem.get(conf);
        addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),
                localResources, null);

        // Set the log4j properties if needed
        if (!log4jPropFile.isEmpty()) {
            addToLocalResources(fs, log4jPropFile, log4jPath, appId.toString(),
                    localResources, null);
        }
    if (!shellCommand.isEmpty()) {
            addToLocalResources(fs, null, shellCommandPath, appId.toString(),
                    localResources, shellCommand);
        }

        if (shellArgs.length > 0) {
            addToLocalResources(fs, null, shellArgsPath, appId.toString(),
                    localResources, StringUtils.join(shellArgs, " "));
        }
    ```
    3) 运行环境(例：hadoop特定的类路径、java classpath、环境变量等)
`Map<String, String> env = new HashMap<String, String>();`
    4) 启动ApplicationMaster的命令(Set the necessary command to execute the application master)
    `Vector<CharSequence> vargs = new Vector<CharSequence>(30);`

* 将验证信息添加到AMContainer，包括RMCredentials以及DockerCredents
```java
       if (rmCredentials != null || dockerCredentials != null) {
            DataOutputBuffer dob = new DataOutputBuffer();
            if (rmCredentials != null) {
                rmCredentials.writeTokenStorageToStream(dob);
            }
            if (dockerCredentials != null) {
                dockerCredentials.writeTokenStorageToStream(dob);
            }
            ByteBuffer tokens = ByteBuffer.wrap(dob.getData(), 0, dob.getLength());
            amContainer.setTokens(tokens);
        }
```

* Client使用ApplicationSubmissionContext提交Application到RM的ASM，并通过按周期向RM请求ApplicationReport，完成对Application的监控。其中，如果Applicaiton运行时间超过timeout限制（默认600000ms，可设置），Client将发送KillApplicationRequest到ResourceManager，将application杀死。
`yarnClient.submitApplication(appContext); // 提交到ASM`
`return monitorApplication(appId); // 监控monitor`

##### 方法功能简介
* main方法将运行init方法，如果init方法返回true则运行run方法。
* init方法解析用户提交的命令，解析用户命令中的参数值。
* run方法完成Client源码逻辑中描述的功能。

#### ApplicationMaster类
##### 源码逻辑
一个AM将在启动一个或者多个Container后，在Container上执行shell命令或脚本。换句话说就是：AM向RM/NM注册回调函数，然后请求Container；得到Container后提交任务，并跟踪这些任务的执行情况，如果失败了则重新提交，直到全部任务完成。

1. RM启动一个Container用于运行AM
2. AM连接RM，向RM注册自己
    * 向RM注册的信息有：
        * AM的port
        * AM所在主机的hostname
        * AM的tracking url。客户端可以用tarcking url来跟踪任务的状态和历史记录
   `RegisterApplicationMasterResponse response = amRMClient
                .registerApplicationMaster(appMasterHostname, appMasterRpcPort,
                        appMasterTrackingUrl, placementConstraintMap);`
    * 需要注意的是：在DistributedShell中，不需要注册tracking url和appMasterRpcPort，只需要设置hostname。
    `private int appMasterRpcPort = -1;`
    `private String appMasterTrackingUrl = "";`
3. ~~*AM会按照设定的时间间隔向RM发送心跳。RM的ApplicationMasterService每次收到ApplicationMaster的心跳信息后，会同时在AMLivelinesMonitor更新其最近一次发送心跳的时间。*~~

4. ApplicationMaster通过amRMClient.addContainerRequest方法向RM发动请求，申请相应数目的Container。在发送申请Container请求前，需要初始化Request，需要初始化的参数有：
    * Priority：请求的优先级
    * capability：当前支持CPU和Memory
    * nodes：申请的Container所在的host（默认null）
    * racks：申请的Container所在的rack（默认null）
```java
        if (this.placementSpecs == null) {
            LOG.info("placementSpecs null");
            for (int i = 0; i < numTotalContainersToRequest; ++i) {
                ContainerRequest containerAsk = setupContainerAskForRM();
                amRMClient.addContainerRequest(containerAsk);
            }
        } else {
            LOG.info("placementSpecs to create req:" + placementSpecs);
            List<SchedulingRequest> schedReqs = new ArrayList<>();
            for (PlacementSpec pSpec : this.placementSpecs.values()) {
                LOG.info("placementSpec :" + pSpec + ", container:" + pSpec
                        .getNumContainers());
                for (int i = 0; i < pSpec.getNumContainers(); i++) {
                    SchedulingRequest sr = setupSchedulingRequest(pSpec);
                    schedReqs.add(sr);
                }
            }
            amRMClient.addSchedulingRequests(schedReqs);
        }
```

5. RM返回AM申请的Containers信息，根据container的状态（containerStatus），更新已申请的成功和还未申请的Container数目。
6. 申请成功的Container，AM则通过ContainerLaunchContext初始化Container的启动信息。初始化container后启动container。需要初始化的信息有：
    * Container ID
    * 执行资源（Shell脚本或命令、处理的数据）
    * 运行环境
    * 运行命令
7. Container运行期间，ApplicationMaster对container进行监控

8. 实现5、6、7的源代码
    * ApplicationMaster类中的一个内部类`RMCallbackHandler`
```java
class RMCallbackHandler extends AMRMClientAsync.AbstractCallbackHandler {
    
    // 处理消息：Containers执行完毕。在RM返回的心跳中携带。如果心跳应答中有已完成和新分配两种Container，先处理已完成的。检查Container执行结果，如果执行失败则重新发起请求直到全部完成。
    @Overrite
    public void onContainersCompleted(List<ContainerStatus> completedContainers){}
    
    // 处理消息：RM新分配Container。在RM返回的心跳中携带。
    // 在createLaunchContainerThread() 方法中使用了LaunchContainerRunnable类。
    @Overrite
    public void onContainersAllocated(List<Container> allocatedContainers) {
        [...]
        // 收到分配的Container后，会提交任务到NM。
        // 新建线程
        Thread launchThread = createLaunchContainerThread(allocatedContaienr, yarnShellId);
        
        launchTheads.add(launchThread);
        launchContainers.add(allocatedContaienr.getId());
        
        // 线程中提交Container到NM，不影响主线程。
        launchThread.start();
        [...]
    }
  
    // 处理消息：Node节点发生变化(health/available etc.)
    @Overtrite
    public void onContainersUpdates(List<UpdatedContainer> containers){}
    
}
```
    
*  `LaunchContainerRunnable`类简析，也是ApplicationMaster的内部类
```java
    private class LaunchContainerRunnable inplements Runable {
        [...]
        public LaunchContainerRunnable(Container lcontainer,
                NMCallbackHandler containerListener, String shellId) {
            this.container = lcontainer; // 创建时记录待使用的Container
            this.containerListener = containerListener;
        }
        
        @Overide
        public void run() {
            // 根据命令、环境变量、本地资源等创建Container加载上下文
            ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
                    localResources, myShellEnv, commands, null, allTokens.duplicate(), null, containerRetryContext);
            // 异步启动Contaienr
            nmClientAsync.startContainerAsync(container, ctx);
                    
        }
        [...]
    }
    
```
    * NM的消息的响应由NMCallbackHandler处理，其也是ApplicationMaster的内部类。
    回调句柄对NM通知过来的各种事件的处理比较简单，只是简单修改AM维护的Container执行完成、失败的个数。等到Container执行完毕后，可以重新发起请求。
```java
class NMCallbackHandler extends NMClientAsync.AbstractCallbackHandler {
    
}
```
     
8. job运行结束，ApplicationMaster~~发起FinishApplicaitionMasterRequest请求~~给RM，完成AM的注销。

##### 方法功能简介
* init方法：完成对执行命令的解析，获取命令中参数指定的值
* run方法：完成ApplicationMaster的启动、注册、container的请求、分配、监控等功能的启动。
    * run方法中建立了与RM通信的Handle-AMRMClientAsync，其中CallbackHandler是由RMCallbackHandle类实现的。
        * RMCallbackHandle类实现了Containers的申请、分配等方法；
        * Containers的分配方法onContainersAllocated中通过LaunchContainerRunnable类中run方法完成Container的启动
* finish方法：完成Container的停止、AM的注销。


##### 对于Yarn来说的的ApplicationMaster(简称为AM)
AM功能总体说来是作业/任务管理。
* AM启动过程
应用程序被提交后，将在RM中申请一个Container来启动引导进程。一旦分配了Container，AM的启动器将直接与AM Container相应的NodeManager通信，以设置并启动Container。
* AM与YARN其余部分的互动简要过程
    1） 应用程序提交一个请求到RM
    2） AM启动并像RM注册
    3） AM像RM请求Container执行实际工作
    4） （？RM分配的）将分配的Container告知NodeManager以便AM使用
    5） 计算过程在Container中进行，将Container与AM（不是RM）保持通信，并告知染污过程。
    6） 应用完成后，Container被停止，AM从RM中注销。
* AM启动后的任务
    1） 初始化向RM报告自己活跃信息的进程
    2） 计算应用程序的资源需求
    3） 将需求转换成YARN调度器可以理解的ResoureRequest
    4） 与调度器协商申请资源
    5） 与NodeManager合作使用分配的Container
    6） 跟踪正在运行的Container的状态，监控它们的进程
    7） 对Container或节点失败的情况进行处理，在必要的情况下重新申请资源