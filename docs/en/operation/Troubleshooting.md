---
layout: global
title: Troubleshooting
nickname: Troubleshooting
group: Operations
priority: 14
---

* Table of Contents
{:toc}

This page is a collection of high-level guides and tips regarding how to diagnose issues encountered in
Alluxio.

Note: this doc is not intended to be the full list of Alluxio questions.
Join the [Alluxio community Slack Channel](https://www.alluxio.io/slack) to chat with users and
developers, or post questions on the [Alluxio Mailing List](https://groups.google.com/forum/#!forum/alluxio-users).

## Where are the Alluxio logs?

Alluxio generates Master, Worker and Client logs under the dir `${ALLUXIO_HOME}/logs`. They are
named as `master.log`, `master.out`, `worker.log`, `worker.out` and `user_${USER}.log`. Files
suffixed with `.log` are generated by log4j; File suffixed with `.out` are generated by redirection of
stdout and stderr of the corresponding process.

The master and worker logs are useful to understand what the Alluxio Master and
Workers are doing, especially when running into any issues. If you do not understand the error messages,
search for them in the [Mailing List](https://groups.google.com/forum/#!forum/alluxio-users),
in the case the problem has been discussed before.

The client-side logs are also helpful when Alluxio service is running but the client cannot connect to the servers.
Alluxio client emits logging messages through log4j, so the location of the logs is determined by the client side
log4j configuration used by the application.

For more information about logging, please check out
[this page]({{ '/en/operation/Basic-Logging.html' | relativize_url }}).

## Alluxio remote debug

Java remote debugging makes it easier to debug Alluxio at the source level without modifying any code. You
will need to append the JVM remote debugging parameters and start a debugging server. There are several ways to append
the remote debugging parameters; you can export the following configuration properties in shell or `alluxio-env.sh`:

```bash
export ALLUXIO_WORKER_JAVA_OPTS="$ALLUXIO_JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=6606"
export ALLUXIO_MASTER_JAVA_OPTS="$ALLUXIO_JAVA_OPTS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=6607"
export ALLUXIO_USER_DEBUG_JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=6609"
```

If you want to debug shell commands, you can add the `-debug` flag to start a debug server with the JVM debug
parameters `ALLUXIO_USER_DEBUG_JAVA_OPTS`, such as `alluxio fs -debug ls /`.

`suspend = y/n` will decide whether the JVM process wait until the debugger connects. If you want to debug with the
shell command, set the `suspend = y`. Otherwise, you can set `suspend = n` to avoid unnecessary waiting time.

After starting the master or worker, use Eclipse, IntelliJ IDEA, or another java IDE. Create a new java remote configuration,
set the debug server's host and port, and start the debug session. If you set a breakpoint which can be reached, the IDE
will enter debug mode and you can inspect the current context's variables, call stack, thread list, and expression
evaluation.

## Alluxio collectInfo command

Alluxio has a `collectInfo` command that collect information to troubleshoot an Alluxio cluster.
`collectInfo` will run a set of sub-commands that each collects one aspect of system information, as explained below.
In the end the collected information will be bundled into one tarball which contains a lot of information regarding your Alluxio cluster.
The tarball size mostly depends on your cluster size and how much information you are collecting.
For example, `collectLog` operation can be costly if you have huge amounts of logs. Other commands
typically do not generate files larger than 1MB. The tarball will help you troubleshoot your cluster.
Or you can shared the tarball with someone you trust to help troubleshoot your Alluxio cluster.

The `collectInfo` command will SSH to each node and execute the set of sub-commands.
In the end of execution the collected information will be written to files and tarballed.
Each individual tarball will be collected to the issuing node.
Then all the tarballs will be bundled into the final tarball, which contains all information about the Alluxio cluster.

>NOTE: Be careful if your configuration contains credentials like AWS keys!
You should ALWAYS CHECK what is in the tarball and REMOVE the sensitive information from the tarball before sharing it with someone!

### Collect Alluxio cluster information
`collectAlluxioInfo` will run a set of Alluxio commands that collect information about the Alluxio cluster, like `bin/alluxio fsadmin report` etc.
When the Alluxio cluster is not running, this command will fail to collect some information.
> NOTE: The configuration parameters will be collected with `alluxio getConf --master`, which obfuscates the credential fields passed to
Alluxio as properties.

### Collect Alluxio configuration files
`collectConfig` will collect all the configuration files under `${alluxio.work.dir}/conf`.
> WARNING: If you put credential fields in the configuration files, DO NOT share the collected tarball with anybody unless
you have manually obfuscated them in the tarball!

### Collect Alluxio logs
`collectLog` will collect all the logs under `${alluxio.work.dir}/logs`.
> NOTE: Roughly estimate how much log you are collecting before executing this command!

### Collect Alluxio metrics
`collectMetrics` will collect Alluxio metrics served at `http://${alluxio.master.hostname}:${alluxio.master.web.port}/metrics/json/` by default.
The metrics will be collected multiple times to see the progress.

### Collect JVM information
`collectJvmInfo` will collect information about the existing JVMs on each node.
This is done by running a `jps` command then `jstack` on each found JVM process.
This will be done multiple times to see if the JVMs are making progress.

### Collect system information
`collectEnv` will run a set of bash commands to collect information about the running node.
This runs system troubleshooting commands like `env`, `hostname`, `top`, `ps` etc.
> WARNING: If you stored credential fields in environment variables like AWS_ACCESS_KEY or in process start parameters
like -Daws.access.key=XXX, DO NOT share the collected tarball with anybody unless you have manually obfuscated them in the tarball!

### Collect all information mentioned above
`all` will run all the sub-commands above.

### Command options

The `collectInfo` command has the below options.

```console
$ bin/alluxio collectInfo [--local] [--max-threads threadNum]
    [all <outputPath>]
    [collectAlluxioInfo <outputPath>]
    [collectConfig <outputPath>]
    [collectEnv <outputPath>]
    [collectJvmInfo <outputPath>]
    [collectLogs <outputPath>]
    [collectMetrics <outputPath>]
```

`<outputPath>` is the directory you want the final tarball to be written into.

Options:
1. `--local` option specifies the `collectInfo` command to run only on `localhost`.
That means the command will only collect information about the `localhost`.

1. `--max-threads threadNum` option configures how many threads to use for concurrently collecting information and transmitting tarballs.
When the cluster has a large number of nodes, or large log files, the network IO for transmitting tarballs can be significant.
Use this parameter to constrain the resource usage of this command.

## Setup FAQ

### Q: I'm new to Alluxio and cannot set up Alluxio on my local machine. What should I do?

A: Check `${ALLUXIO_HOME}/logs` to see if there are any master or worker logs. Look for any errors
in these logs. Double check if you missed any configuration
steps in [Running-Alluxio-Locally]({{ '/en/deploy/Running-Alluxio-Locally.html' | relativize_url }}).

Typical issues:
- `ALLUXIO_MASTER_MOUNT_TABLE_ROOT_UFS` is not configured correctly.
- If running `ssh localhost` fails, make sure the public SSH key for the host is added in `~/.ssh/authorized_keys`.

### Q: I'm trying to deploy Alluxio in a cluster with Spark and HDFS. Are there any suggestions?

A: Please follow [Running-Alluxio-on-a-Cluster]({{ '/en/deploy/Running-Alluxio-On-a-Cluster.html' | relativize_url }}),
[Configuring-Alluxio-with-HDFS]({{ '/en/ufs/HDFS.html' | relativize_url }}).

Tips:

- The best performance gains occur when Alluxio workers are co-located with the nodes of the computation frameworks.
- You can use Mesos and Yarn integration if you are already using Mesos or Yarn to manage your cluster.
- If the under storage is remote (like S3 or remote HDFS), using Alluxio can be especially beneficial.

### Q: Why do I see "Unsupported major.minor version 52.0" error when I start Alluxio?

A: Alluxio requires Java 8 runtime to function properly. If this error is seen at the start of Alluxio master orworker, please setup
your environment so that the default Java version is 8. If you see this error while using an application to access files on Alluxio,
please make sure your application is running on Java 8.

## Usage FAQ

### Q: Why do I see exceptions like "No FileSystem for scheme: alluxio"?

A: This error message is seen when your applications (e.g., MapReduce, Spark) try to access
Alluxio as an HDFS-compatible file system, but the `alluxio://` scheme is not recognized by the
application. Please make sure your HDFS configuration file `core-site.xml` (in your default hadoop
installation or `spark/conf/` if you customize this file for Spark) has the following property:

```xml
<configuration>
  <property>
    <name>fs.alluxio.impl</name>
    <value>alluxio.hadoop.FileSystem</value>
  </property>
</configuration>
```

### Q: Why do I see exceptions like "java.lang.RuntimeException: java.lang.ClassNotFoundException: Class alluxio.hadoop.FileSystem not found"?

A: This error message is seen when your applications (e.g., MapReduce, Spark) try to access
Alluxio as an HDFS-compatible file system, the `alluxio://` scheme has been
configured correctly, but the Alluxio client jar is not found on the classpath of your application.
Depending on the computation frameworks, users usually need to add the Alluxio
client jar to their class path of the framework through environment variables or
properties on all nodes running this framework. Here are some examples:

- For MapReduce jobs, you can append the client jar to `$HADOOP_CLASSPATH`:

```console
$ export HADOOP_CLASSPATH={{site.ALLUXIO_CLIENT_JAR_PATH}}:${HADOOP_CLASSPATH}
```

- For Spark jobs, you can append the client jar to `$SPARK_CLASSPATH`:

```console
$ export SPARK_CLASSPATH={{site.ALLUXIO_CLIENT_JAR_PATH}}:${SPARK_CLASSPATH}
```

Alternatively, add the following lines to `spark/conf/spark-defaults.conf`:

```properties
spark.driver.extraClassPath {{site.ALLUXIO_CLIENT_JAR_PATH}}
spark.executor.extraClassPath {{site.ALLUXIO_CLIENT_JAR_PATH}}
```

- For Presto, put Alluxio client jar `{{site.ALLUXIO_CLIENT_JAR_PATH}}` into the directory
`${PRESTO_HOME}/plugin/hive-hadoop2/`
Since Presto has long running processes, ensure they are restarted after the jar has been added.

- For Hive, set `HIVE_AUX_JARS_PATH` in `conf/hive-env.sh`:

```console
$ export HIVE_AUX_JARS_PATH={{site.ALLUXIO_CLIENT_JAR_PATH}}:${HIVE_AUX_JARS_PATH}
```
Since Hive has long running processes, ensure they are restarted after the jar has been added.

If the corresponding classpath has been set but exceptions still exist, users can check
whether the path is valid by:

```console
$ ls {{site.ALLUXIO_CLIENT_JAR_PATH}}
```

### Q: I'm seeing error messages like "Frame size (67108864) larger than max length (16777216)". What is wrong?

A: This problem can be caused by different possible reasons.

- Please double check if the port of Alluxio master address is correct. The default listening port for Alluxio master is port 19998,
while a common mistake causing this error message is due to using a wrong port in master address (e.g., using port 19999 which is the default Web UI port for Alluxio master).
- Please ensure that the security settings of Alluxio client and master are consistent.
Alluxio provides different approaches to [authenticate]({{ '/en/operation/Security.html' | relativize_url }}#authentication) users by configuring `alluxio.security.authentication.type`.
This error happens if this property is configured with different values across servers and clients
(e.g., one uses the default value `NOSASL` while the other is customized to `SIMPLE`).
Please read [Configuration-Settings]({{ '/en/operation/Configuration.html' | relativize_url }}) for how to customize Alluxio clusters and applications.

### Q: I'm copying or writing data to Alluxio while seeing error messages like "Failed to cache: Not enough space to store block on worker". Why?

A: This error indicates insufficient space left on Alluxio workers to complete your write request.

- Check if you have any files unnecessarily pinned in memory and unpin them to release space.
See [Command-Line-Interface]({{ '/en/operation/User-CLI.html' | relativize_url }}) for more details.
- Increase the capacity of workers by changing `alluxio.worker.memory.size` property.
See [Configuration]({{ '/en/reference/Properties-List.html' | relativize_url }}#common-configuration) for more description.

### Q: I'm writing a new file/directory to Alluxio and seeing journal errors in my application

A: When you see errors like "Failed to replace a bad datanode on the existing pipeline due to no more good datanodes being available to try",
it is because Alluxio master failed to update journal files stored in a HDFS directory according to
the property `alluxio.master.journal.folder` setting. There can be multiple reasons for this type of errors, typically because
some HDFS datanodes serving the journal files are under heavy load or running out of disk space. Please ensure the
HDFS deployment is connected and healthy for Alluxio to store journals when the journal directory is set to be in HDFS.

### Q: I added some files in under file system. How can I reveal the files in Alluxio?

By default, Alluxio loads the list of files the first time a directory is visited.
Alluxio will keep using the cached file list regardless of the changes in the under file system.
To reveal new files from under file system, you can use the command
`alluxio fs ls -R -Dalluxio.user.file.metadata.sync.interval=${SOME_INTERVAL} /path` or by setting the same
configuration property in masters' `alluxio-site.properties`.
The value for the configuration property is used to determine the minimum interval between two syncs.
You can read more about loading files from underfile system
[here]({{ '/en/core-services/Unified-Namespace.html' | relativize_url }}#ufs-metadata-sync).

### Q: I see an error "Block ?????? is unavailable in both Alluxio and UFS" while reading some file. Where is my file?

A: When writing files to Alluxio, one of the several write type can be used to tell Alluxio worker how the data should be stored:

`MUST_CACHE`: data will be stored in Alluxio only

`CACHE_THROUGH`: data will be cached in Alluxio as well as written to UFS

`THROUGH`: data will be only written to UFS

`ASYNC_THROUGH`: data will be stored in Alluxio synchronously and then written to UFS asynchronously

By default the write type used by Alluxio client is `ASYNC_THROUGH`, therefore a new file written to Alluxio is only stored in Alluxio
worker storage, and can be lost if a worker crashes. To make sure data is persisted, either use `CACHE_THROUGH` or `THROUGH` write type,
or increase `alluxio.user.file.replication.durable` to an acceptable degree of redundancy.

Another possible cause for this error is that the block exists in the file system, but no worker has connected to master. In that
case the error will go away once at least one worker containing this block is connected.

### Q: I'm running an Alluxio shell command and it hangs without giving any output. What's going on?

A: Most Alluxio shell commands require connecting to Alluxio master to execute. If the command fails to connect to master it will
keep retrying several times, appearing as "hanging" for a long time. It is also possible that some command can take a long time to
execute, such as persisting a large file on a slow UFS. If you want to know what happens under the hood, check the user log (stored
as `${ALLUXIO_HOME}/logs/user_${USER_NAME}.log` by default) or master log (stored as `${ALLUXIO_HOME}/logs/master.log` on the master
node by default).

## Performance FAQ

### Q: I tested Alluxio/Spark against HDFS/Spark (running simple word count of GBs of files). Why is there no discernible performance difference?

A: Alluxio accelerates your system performance by leveraging temporal or spatial locality using distributed in-memory storage
(and tiered storage). If your workloads don't have any locality, you will not see noticeable performance boost.

**For a comprehensive guide on tuning performance of Alluxio cluster, please check out [this page]({{ '/en/operation/Performance-Tuning.html' | relativize_url }}).**

## Environment

Alluxio can be configured under a variety of modes, in different production environments.
Please make sure the Alluxio version being deployed is update-to-date and supported.

When posting questions on the [Mailing List](https://groups.google.com/forum/#!forum/alluxio-users),
please attach the full environment information, including
- Alluxio version
- OS version
- Java version
- UnderFileSystem type and version
- Computing framework type and version
- Cluster information, e.g. the number of nodes, memory size in each node, intra-datacenter or cross-datacenter