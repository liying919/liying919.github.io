---
layout: post
title:  "Flink Job Detached Mode Issue"
date:   2020-01-17 15:03:36 +0800
categories: Flink BigData
---
Previously we use blocking mode to deploy jars on YARN, they run well. Recently we find the client process occupies more and more memory , so we try to use detached mode, but the job failed to deploy with following error information:

```
The program finished with the following exception:

org.apache.flink.client.deployment.ClusterDeploymentException: Could not deploy Yarn job cluster.
        at org.apache.flink.yarn.YarnClusterDescriptor.deployJobCluster(YarnClusterDescriptor.java:82)
        at org.apache.flink.client.cli.CliFrontend.runProgram(CliFrontend.java:230)
        at org.apache.flink.client.cli.CliFrontend.run(CliFrontend.java:205)
        at org.apache.flink.client.cli.CliFrontend.parseParameters(CliFrontend.java:1010)
        at org.apache.flink.client.cli.CliFrontend.lambda$main$10(CliFrontend.java:1083)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1754)
        at org.apache.flink.runtime.security.HadoopSecurityContext.runSecured(HadoopSecurityContext.java:41)
        at org.apache.flink.client.cli.CliFrontend.main(CliFrontend.java:1083)
Caused by: org.apache.flink.yarn.AbstractYarnClusterDescriptor$YarnDeploymentException: The YARN application unexpectedly switched to state FAILED during deployment.
Diagnostics from YARN: Application application_1533815330295_30183 failed 2 times due to AM Container for appattempt_xxxx exited with  exitCode: 1
For more detailed output, check application tracking page:http:xxxxThen, click on links to logs of each attempt.
Diagnostics: Exception from container-launch.
Container id: container_e05_xxxx
Exit code: 1
Stack trace: ExitCodeException exitCode=1:
        at org.apache.hadoop.util.Shell.runCommand(Shell.java:593)
        at org.apache.hadoop.util.Shell.run(Shell.java:490)
        at org.apache.hadoop.util.Shell$ShellCommandExecutor.execute(Shell.java:784)
        at org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor.launchContainer(LinuxContainerExecutor.java:298)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:324)
        at org.apache.hadoop.yarn.server.nodemanager.containermanager.launcher.ContainerLaunch.call(ContainerLaunch.java:83)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)

Shell output: main : command provided 1
main : user is streams
main : requested yarn user is user1
```

After asking in [flink@apache][flink@apache], got an detailed answer by Yang Wang:

>> Why the Yarn per-job attach mode could work, but detach mode could not?
It is just becausein 1.9 and previous versions, the per-job have very different code path for attach and detach
mode. For attach mode, Flink client will start a session cluster, and then submit a job to the existing session.
So all the users jars are loaded by user classloader, not system classloader. For detach mode, all the jars will
be shipped by Yarn local resources and appended to the system classpath of jobmanager and taskmanager.
The behavior will be changed from 1.10. Both detach and attach will always be the real per-job, not simulate
by session. You could check FLIP-82 for more information[1].

>> How to fix this problem?
1. If you Yarn cluster could support multiple hdfs clusters, then you will not need to add hdfs configuration in
you jar. That's how we use it in production environment.
2. If you can not change this, and you will use Flink 1.10. Then you could set 
`yarn.per-job-cluster.include-user-jar: DISABLED`. Then all the user jars will not be added to system classpath.
Instead, they will be loaded by user classloader. This is a new feature in 1.10. Check more information here[2].
3. If you are still using the 1.9 and previous versions, move the hdfs configuration out of your jar. Then use `-t`
to ship your hadoop configuration and reset the hadoop env.
-yt /path/of/my-hadoop-conf
-yD containerized.master.env.HADOOP_CONF_DIR='$PWD/my-hadoop-conf'
-yD containerized.taskmanager.env.HADOOP_CONF_DIR='$PWD/my-hadoop-conf'


[1].https://cwiki.apache.org/confluence/display/FLINK/FLIP-82%3A+Use+real+per-job+mode+for+YARN+per-job+attached+execution
[2].https://issues.apache.org/jira/browse/FLINK-13993

[flink@apache]: user@flink.apache.org
