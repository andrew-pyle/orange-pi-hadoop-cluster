## Troubleshooting the Cluster
At this point, I have no idea what is malfunctioning with this cluster. The reported cause of a MapReduce job failing seems to be different every time!

I'll try to collect all the logs I can for analysis, gather some more tutorials, and troubleshoot the namenode. Then I'll clone the OS and restart the cluster.

> NOTE: The Hadoop logging protocol is very complicated, and after several hours of attempting to find the cause of the job failure, I decided that it is not worth spending even more time to discover the root cause of the failure. I'll just troubleshoot "from the ground up"

Here's what I tried:

### Ensure Hostnames Working Properly
Include the Hadoop cluster ndoes in my client machine's (MacBook Pro) `etc/hosts` file:
```bash
$ sudo nano /etc/hosts
```
Add the Hadoop nodes:
```bash
### Hadoop Cluster Nodes - Orange Pi ###
192.168.0.110	hadoopnode1
192.168.0.111	hadoopnode2
```
This now works from my client machine:
```bash
$ ssh hduser@hadoopnode1
```

### Check the Java version
Check to see if the java version is correct:
```bash
$ java -version

java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) Client VM (build 25.121-b13, mixed mode)
```
Java 1.8.0 seems to be the version I installed, so that looks right.

### Check Hostnames
```bash
$ cat /etc/hosts

127.0.0.1   localhost orangepione
::1         localhost orangepione ip6-localhost ip6-loopback
fe00::0     ip6-localnet
ff00::0     ip6-mcastprefix
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
192.168.0.110 hadoopnode1
192.168.0.111 hadoopnode2
```
```bash
$ cat /etc/hostname

hadoopnode1

```
There was an extra newline after `hadoopnode1`, so I deleted that.

### SSH Access
```bash
$ cat ~/.ssh/authorized_keys

ssh-rsa <KEY REDACTED FOR SECURITY> hduser@orangepione
```
Aha! We might be on to something here. The SSH key is for hduser@orangepione, which doesn't exist anymore. We changed the hostname from orangepione to hadoopnode1/2.

Let's delete the `~/.ssh` directory and recreate the SSH key:
```bash
$ ssh-keygen -t rsa -P ""
$ cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys
# Replace    ^^^^^^^^^^ with the directory you choose
# in the ssh-keygen command. id_rsa is default.
```
Copying the `id_rsa.pub` (public key) into the `authorized_keys` file allows any machine with the `id_rsa` (private key) to login without a password.

[Rsync](https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories-on-a-vps) is a great way to clone the folder to the home folder of the other nodes:
```bash
$ rsync -av ~/.ssh hduser@hadoopnode2:~ # sync the .ssh folder to the ~ directory of the other node.
```

Ensure the SSH key allows passwordless login:
```bash
$ ssh hduser@hadoopnode1 # It works! (or not)
$ exit
```

*Update: Deleting the hostname after the public key in `id_rsa.pub` and in `authorized_keys` did not change the ability to SSH passwordless, so that is probably not the issue.*

### Hadoop Install Verification
```bash
$ hadoop version

Hadoop 2.7.3
Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r baa91f7c6bc9cb92be5982de4719c1c8af91ccff
Compiled by root on 2016-08-18T01:41Z
Compiled with protoc 2.5.0
From source with checksum 2e4ce5f957ea4db193bce3734ff29ff4
This command was run using /opt/hadoop-2.7.3/share/hadoop/common/hadoop-common-2.7.3.jar
```
Seems to be Hadoop 2.7.3, so that's right.

### HDFS `fsck`

I ran the Hadoop filesystem check:
```bash
$ hdfs fsck / # Check the whole dfs, starting from /

..................Status: HEALTHY
 Total size:	1508057 B
 Total dirs:	11
 Total files:	21
 Total symlinks:		0
 Total blocks (validated):	21 (avg. block size 71812 B)
 Minimally replicated blocks:	21 (100.0 %)
 Over-replicated blocks:	0 (0.0 %)
 Under-replicated blocks:	2 (9.523809 %)
 Mis-replicated blocks:		0 (0.0 %)
 Default replication factor:	2
 Average block replication:	2.0
 Corrupt blocks:		0
 Missing replicas:		16 (27.586206 %)
 Number of data-nodes:		2
 Number of racks:		1
FSCK ended at Sat Apr 22 11:44:53 CDT 2017 in 126 milliseconds


The filesystem under path '/' is HEALTHY
```
It looks like corrupted blocks is not the issue. I'm guessing the 2 `under-replicated blocks` are due to failed jobs.

### Reformat HDFS
I saved all the HDFS files to my client machine:
```bash
$ hdfs dfs -get / ~/HdfsDump
$ scp -r HdfsDump user@clientipaddress:~

$ scp -r /hdfs user@clientipaddress:~
```
Delete all HDFS files and reformat:
```bash
$ rm -rf /hdfs/*
$ hdfs namenode -format
```
```bash
$ hdfs dfsadmin -report
```
After starting the `dfs` and `yarn`, only `hadoopnode1` is connected. This may be due to a cluster ID conflict.

I'll use the hdfs directory structure from [Jason Carter's blog](https://medium.com/@jasonicarter/how-to-hadoop-at-home-with-raspberry-pi-part-3-7d114d35fdf1) (parts 2 & 3 specifically).

I'll remove all the files on both nodes and try again:
```bash
$ sudo rm -rf /hdfs # hadoopnode1 & 2
$ sudo mkdir -p /hdfs/namenode # only hadoopnode1
$ sudo mkdir -p /hdfs/datanode # hadoopnode1 & 2
$ sudo chown hduser:hadoop /hdfs/ -R # change directory ownership
$ chmod 750 /hdfs # change directory permissions
$ ls -l / | grep hdfs # view directory

drwxr-x---  4 hduser hadoop  4096 Apr 22 12:57 hdfs
```
Change configuration to  new `/hdfs` directory structure:
```xml
File: hdfs-site.xml

<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>5242880</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.name.dir</name>
    <value>file:/hdfs/datanode</value>
  </property>
</configuration>
```
Sync to `hadoopnode2`:
```bash
$ rsync -av $HADOOP_CONF_DIR/ hduser@hadoopnode2:$HADOOP_CONF_DIR
# rsync -anv will do a dry-run so you can see the specific files
# to be sent before you actually send anything.
```
Create HDFS again:
```bash
# On the namenode (hadoopnode1 for me)
$ hdfs namenode -format

$ $HADOOP_HOME/sbin/start-dfs.sh
$ $HADOOP_HOME/sbin/start-yarn.sh
```
Check the status:
```bash
$ hdfs dfsadmin -report

Configured Capacity: 30753898496 (28.64 GB)
Present Capacity: 26473508864 (24.66 GB)
DFS Remaining: 26473459712 (24.66 GB)
DFS Used: 49152 (48 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (2):

Name: 192.168.0.111:50010 (hadoopnode2)
Hostname: hadoopnode2
Decommission Status : Normal
Configured Capacity: 15376949248 (14.32 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 2121494528 (1.98 GB)
DFS Remaining: 13255430144 (12.35 GB)
DFS Used%: 0.00%
DFS Remaining%: 86.20%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Apr 22 13:18:48 CDT 2017


Name: 192.168.0.110:50010 (hadoopnode1)
Hostname: hadoopnode1
Decommission Status : Normal
Configured Capacity: 15376949248 (14.32 GB)
DFS Used: 24576 (24 KB)
Non DFS Used: 2158895104 (2.01 GB)
DFS Remaining: 13218029568 (12.31 GB)
DFS Used%: 0.00%
DFS Remaining%: 85.96%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Sat Apr 22 13:18:50 CDT 2017
```
Both `datanodes` up!

### Hadoop Configuration
I'm going to try the Hadoop configuration from [Jason Carter's blog](https://medium.com/@jasonicarter/how-to-hadoop-at-home-with-raspberry-pi-part-3-7d114d35fdf1) (parts 2 & 3 specifically), as we did above.
1. `yarn-site.xml`
```xml
<configuration>
<!-- Site specific YARN configuration properties -->
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoopnode1</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
</configuration>
```
2. `hdfs-site.xml`
```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>5242880</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.name.dir</name>
    <value>file:/hdfs/datanode</value>
  </property>
</configuration>
```
3. `mapred-site.xml`
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```
4. `core-site.xml`
```xml
<configuration>
 <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoopnode1:53410</value>
  </property>
</configuration>
```
>I noticed here that I left the following setting in `core-site.xml` when I reformatted the HDFS:
>```xml
><property>
>  <name>hadoop.tmp.dir</name>
>  <value>/hdfs/tmp/</value>
></property>
>```
> I should probably reformat the HDFS again to make it comply with the new structure.

5. `hadoop-env.sh`
Uncomment the following `export` line and define the HADOOP_HEAPSIZE parameter.
```bash
# The maximum amount of heap to use, in MB. Default is 1000.
export HADOOP_HEAPSIZE=128
```
Sync to `hadoopnode2`:
```bash
$ rsync -av $HADOOP_CONF_DIR/ hduser@hadoopnode2:$HADOOP_CONF_DIR

# hdfs-site.xml won't be sent because we didn't make any new changes since the HDFS reformat ones.
sending incremental file list
core-site.xml
hadoop-env.sh
mapred-site.xml
yarn-site.xml

sent 2,369 bytes  received 182 bytes  1,020.40 bytes/sec
total size is 78,384  speedup is 30.73
```
Reformat HDFS:
```bash
$ $HADOOP_HOME/sbin/stop-dfs.sh
$ $HADOOP_HOME/sbin/stop-yarn.sh
```
```bash
$ sudo rm -rf /hdfs
$ sudo mkdir -p /hdfs/namenode # only hadoopnode1
$ sudo mkdir -p /hdfs/datanode # hadoopnode1 & 2
$ sudo chown hduser:hadoop /hdfs/ -R # change directory ownership
$ chmod 750 /hdfs # change directory permissions
$ ls -l / | grep hdfs # view directory

drwxr-x---  4 hduser hadoop  4096 Apr 22 14:08 hdfs
```
Create HDFS again (again):
```bash
# On the namenode (hadoopnode1 for me)
$ hdfs namenode -format

$ $HADOOP_HOME/sbin/start-dfs.sh
$ $HADOOP_HOME/sbin/start-yarn.sh
```
Check the status:
```bash
$ hdfs dfsadmin -report
```
Both `datanodes` up, and no `/tmp` directory:
```bash
# On namenode
$ ls /hdfs
datanode  namenode
```

### Word Count Test
Load a `.txt` file from the client machine and put it into HDFS:
```bash
$ hdfs dfs -put /path/to/local/file.txt /path/to/HDFS/file.txt
```
Run a wordcount job from the Hadoop example MapReduce applications.
```bash
$ cd /opt/hadoop-2.7.3/share/hadoop/mapreduce
$ ls

hadoop-mapreduce-client-app-2.7.3.jar
hadoop-mapreduce-client-common-2.7.3.jar
hadoop-mapreduce-client-core-2.7.3.jar
hadoop-mapreduce-client-hs-2.7.3.jar
hadoop-mapreduce-client-hs-plugins-2.7.3.jar
hadoop-mapreduce-client-jobclient-2.7.3.jar
hadoop-mapreduce-client-jobclient-2.7.3-tests.jar
hadoop-mapreduce-client-shuffle-2.7.3.jar
hadoop-mapreduce-examples-2.7.3.jar
lib
lib-examples
sources

# Output directory in command must not exist! The job will
# fail if the specified directory exists already.
$ yarn jar hadoop-mapreduce-examples-2.7.3.jar wordcount /path/to/hdfs/file.txt /path/to/output/directory
```
#### SUCCESS!
**Whoop!** CLI Output of first successful Job:
```
Java HotSpot(TM) Client VM warning: You have loaded library /opt/hadoop-2.7.3/lib/native/libhadoop.so.1.0.0 which might have disabled stack guard. The VM will try to fix the stack guard now.
It's highly recommended that you fix the library with 'execstack -c <libfile>', or link it with '-z noexecstack'.
17/04/22 14:25:45 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
17/04/22 14:25:50 INFO client.RMProxy: Connecting to ResourceManager at hadoopnode1/192.168.0.110:8032
17/04/22 14:25:56 INFO input.FileInputFormat: Total input paths to process : 1
17/04/22 14:25:56 INFO mapreduce.JobSubmitter: number of splits:1
17/04/22 14:25:57 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1492888277723_0001
17/04/22 14:25:59 INFO impl.YarnClientImpl: Submitted application application_1492888277723_0001
17/04/22 14:26:00 INFO mapreduce.Job: The url to track the job: http://hadoopnode1:8088/proxy/application_1492888277723_0001/
17/04/22 14:26:00 INFO mapreduce.Job: Running job: job_1492888277723_0001
17/04/22 14:26:39 INFO mapreduce.Job: Job job_1492888277723_0001 running in uber mode : false
17/04/22 14:26:39 INFO mapreduce.Job:  map 0% reduce 0%
17/04/22 14:27:10 INFO mapreduce.Job:  map 100% reduce 0%
17/04/22 14:27:35 INFO mapreduce.Job:  map 100% reduce 100%
17/04/22 14:27:36 INFO mapreduce.Job: Job job_1492888277723_0001 completed successfully
17/04/22 14:27:37 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=349637
		FILE: Number of bytes written=937211
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=1111292
		HDFS: Number of bytes written=257189
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=27800
		Total time spent by all reduces in occupied slots (ms)=21129
		Total time spent by all map tasks (ms)=27800
		Total time spent by all reduce tasks (ms)=21129
		Total vcore-milliseconds taken by all map tasks=27800
		Total vcore-milliseconds taken by all reduce tasks=21129
		Total megabyte-milliseconds taken by all map tasks=28467200
		Total megabyte-milliseconds taken by all reduce tasks=21636096
	Map-Reduce Framework
      Map input records=19150
  		Map output records=184794
  		Map output bytes=1819171
  		Map output materialized bytes=349637
  		Input split bytes=99
  		Combine input records=184794
  		Combine output records=23648
  		Reduce input groups=23648
  		Reduce shuffle bytes=349637
  		Reduce input records=23648
  		Reduce output records=23648
  		Spilled Records=47296
  		Shuffled Maps =1
  		Failed Shuffles=0
  		Merged Map outputs=1
  		GC time elapsed (ms)=1003
  		CPU time spent (ms)=16010
  		Physical memory (bytes) snapshot=215564288
  		Virtual memory (bytes) snapshot=642580480
  		Total committed heap usage (bytes)=131350528
  	Shuffle Errors
  		BAD_ID=0
  		CONNECTION=0
  		IO_ERROR=0
  		WRONG_LENGTH=0
  		WRONG_MAP=0
  		WRONG_REDUCE=0
  	File Input Format Counters
  		Bytes Read=1111193
  	File Output Format Counters
  		Bytes Written=257189
```
Let's look at the actual wordcount output. The output directory has two parts. ([See here for a StackOverflow question about it](http://stackoverflow.com/questions/10666488/what-are-success-and-part-r-00000-files-in-hadoop/10666874#10666874))
1. `_SUCCESS` An empty file that signifies a successful job completion
1. Raw output files
    * `part-x-00000` Output of task number 00000
    * `part-x-00001` Output of task number 00001
    ...
    * `part-x-yyyyy` Output of task number yyyyy

`IliadOutput` contents:
```bash
$ hdfs dfs -ls IliadOutput

Found 2 items
# Signifies successful job
-rw-r--r--   2 hduser supergroup          0 2017-04-22 14:27 /IliadCount/_SUCCESS

# Wordcount output file of the only reduce task
-rw-r--r--   2 hduser supergroup     257189 2017-04-22 14:27 /IliadCount/part-r-00000
```

### Calculate π Test
With the previous configurations, we tried the pi MapReduce example application. Let's try it again with the new configuration.

```bash
$ cd /opt/hadoop-2.7.3/share/hadoop/mapreduce
$ ls

hadoop-mapreduce-client-app-2.7.3.jar
hadoop-mapreduce-client-common-2.7.3.jar
hadoop-mapreduce-client-core-2.7.3.jar
hadoop-mapreduce-client-hs-2.7.3.jar
hadoop-mapreduce-client-hs-plugins-2.7.3.jar
hadoop-mapreduce-client-jobclient-2.7.3.jar
hadoop-mapreduce-client-jobclient-2.7.3-tests.jar
hadoop-mapreduce-client-shuffle-2.7.3.jar
hadoop-mapreduce-examples-2.7.3.jar
lib
lib-examples
sources

$ yarn jar hadoop-mapreduce-examples-2.7.3.jar pi 16 1000
```
The job got stuck here for about 30 minutes.
```bash
INFO mapreduce.Job: Running job: job_1492888277723_0002
```
I think the resources of the Orange Pi are too small for the job  parameters. The touble is that I cannot find any description of the parameters for the pi program anywhere.

I'll try to reconfigure Hadoop to understand the resource limitations of the Orange Pi One.

### Re(RE)(RE)configure
1. `yarn-site.xml`
```xml
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoopnode1</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>384</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>128</value>
  </property>
  <property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>384</value>
  </property>
  <property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
    <description>Whether virtual memory limits will be enforced  for containers.</description>
  </property>
</configuration>
```
2. `mapred-site.xml`
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.map.memory.mb</name>
     <value>256</value>
  </property>
  <property>
     <name>mapreduce.reduce.memory.mb</name>
     <value>384</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.resource.mb</name>
    <value>128</value>
  </property>
</configuration>
```
Propagate the changes with `rsync` as above.
> The $HADOOP_HOME/sbin/start-dfs.sh script has started failing to start the DataNode service. Maybe reformatting the HDFS as above will correct the error.

#### Test
##### Calculate Pi

The application failed with the following messages:

`hadoopnode2`
```
Diagnostics: Container [pid=3907,containerID=container_1492900593461_0001_01_000001] is running beyond virtual memory limits. Current usage: 16.7 MB of 128 MB physical memory used; 1.1 GB of 268.8 MB virtual memory used. Killing container.
```
`hadoopnode1`
```
ApplicationMaster for attempt appattempt_1492900593461_0001_000002 timed out
```
It seems that removing the virtual memory limit enforcement setting was not properly synced to `hadoopnode2`, and the container was killed for exceeding the allotment, which is (2.1 *  physical memory allotment). It also seems that the retry attempt on `hadoopnode1` timed out because the ApplicationMaster could not start a container. This seems to indicate insufficient memory settings.

##### Wordcount
1.1 MB text file from [Project Gutenberg](http://www.gutenberg.org/)

###### Test Job 1:
`hadoopnode2`
```
YarnException: Unauthorized request to start container
```
`hadoopnode1`
```
ApplicationMaster for attempt appattempt_1492900593461_0002_000001 timed out
```
Once again, it seems that insufficient resources are to blame. The container could not start properly.

###### Test Job 2:
Success!
```
17/04/23 07:38:31 INFO client.RMProxy: Connecting to ResourceManager at hadoopnode1/192.168.0.110:8032
17/04/23 07:38:35 INFO input.FileInputFormat: Total input paths to process : 1
17/04/23 07:38:36 INFO mapreduce.JobSubmitter: number of splits:1
17/04/23 07:38:36 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1492900593461_0003
17/04/23 07:38:38 INFO impl.YarnClientImpl: Submitted application application_1492900593461_0003
17/04/23 07:38:38 INFO mapreduce.Job: The url to track the job: http://hadoopnode1:8088/proxy/application_1492900593461_0003/
17/04/23 07:38:38 INFO mapreduce.Job: Running job: job_1492900593461_0003
17/04/23 07:39:18 INFO mapreduce.Job: Job job_1492900593461_0003 running in uber mode : false
17/04/23 07:39:18 INFO mapreduce.Job:  map 0% reduce 0%
17/04/23 07:39:41 INFO mapreduce.Job:  map 67% reduce 0%
17/04/23 07:39:44 INFO mapreduce.Job:  map 100% reduce 0%
17/04/23 07:40:09 INFO mapreduce.Job:  map 100% reduce 100%
17/04/23 07:40:10 INFO mapreduce.Job: Job job_1492900593461_0003 completed successfully
17/04/23 07:40:11 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=349637
		FILE: Number of bytes written=937165
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=1111292
		HDFS: Number of bytes written=257189
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=45734
		Total time spent by all reduces in occupied slots (ms)=62559
		Total time spent by all map tasks (ms)=22867
		Total time spent by all reduce tasks (ms)=20853
		Total vcore-milliseconds taken by all map tasks=22867
		Total vcore-milliseconds taken by all reduce tasks=20853
		Total megabyte-milliseconds taken by all map tasks=5853952
		Total megabyte-milliseconds taken by all reduce tasks=8007552
	Map-Reduce Framework
		Map input records=19150
		Map output records=184794
		Map output bytes=1819171
		Map output materialized bytes=349637
		Input split bytes=99
		Combine input records=184794
		Combine output records=23648
		Reduce input groups=23648
		Reduce shuffle bytes=349637
		Reduce input records=23648
		Reduce output records=23648
		Spilled Records=47296
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=1192
		CPU time spent (ms)=13260
		Physical memory (bytes) snapshot=220286976
		Virtual memory (bytes) snapshot=641531904
		Total committed heap usage (bytes)=132689920
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=1111193
	File Output Format Counters
		Bytes Written=257189
```
This time, it seems like the job stayed within the memory allocation limits. The job only took 1 min 30 sec.

### Discussion

*DISCLAIMER: I do not understand everything about physical and virtual memory management in operating systems, especially under Hadoop, YARN and MapReduce. However, I know that Hadoop was designed for large-scale analytics on machines with many times the resources available in my cluster. In some ways this project was a lost cause before it began. This cluster of Orange Pi Ones is not the environment Hadoop was designed for, indeed, the jobs that I ran in this tutorial could have been run easily on my client machine alone, even with an application written in an interpreted language. But this does not prevent us from improving the setup as much as we can!*


#### Hadoop Application Failure Analysis
Over the course of this project, Hadoop distributed applications failed for 2 main reasons:

1. Application timed out while ApplicationMaster was allocating a container
1. Containers ran beyond virtual memory limits
1. Block could not be found in HDFS

>*I think the "Block could not be found in HDFS" error was due to file corruption after a hard poweroff. HDFS seems to corrupt very easily, so be sure not to pull the plug on a running Hadoop application!*

After doing some research, I think the first two errors are due to the same issue: limited memory resources of the Orange Pi One (512 MB). My thought is that a container was allocated to a given node, and a certain map or reduce task was assigned to that container. During the creation of the container or during the execution of the task, the memory required greatly exceeded the available physical memory, resulting in large virtual memory usage. For example, jobs which failed because of killed containers reported:

* **Physical:** 16.7 MB / 128 MB
* **Virtual:** 1.1 GB / 268.8 MB

This heavy reliance on virtual memory usage caused nodes in use to become unresponsive, causing the application to fail: If virtual memory limits were enforced, the container was killed, or if they were not enforced, the application would become unresponsive, and Hadoop would fail the job.

The configuration resources I used as a guide were either designed for a [production system](https://hortonworks.com/blog/how-to-plan-and-configure-yarn-in-hdp-2-0/) with RAM on the order of 50 GB, or a [Raspberry Pi cluster](http://www.widriksson.com/raspberry-pi-2-hadoop-2-cluster/#YARN_and_MapReduce_memory_configuration_overview) with about double the memory per machine. I attempted to scale down the YARN container allocation, JVM heap size, and Map & Reduce task allocations, but I now think that the extremely small allocations only contributed to the runaway virtual memory usage.

#### Future Work
Future work should address the node resource bottleneck. One concern is distributing the workload among the nodes. On jobs which used only a single task at a time, only one node at a time was involved in the processing at a time. It would certainly seem that Hadoop should be able to allocate smaller map and reduce tasks so that they fit within the memory of the cluster node.

Trying to spread out the file over the HDFS in a more even manner would seem to be a helpful as well. Decreasing the block size from the default value as I have done here seems to be helpful, but not enough to prevent Hadoop from storing all blocks of a small file on one node. [Jonas Widriksson's original Hadoop v1 tutorial](http://www.widriksson.com/raspberry-pi-hadoop-cluster/) has some useful ideas in this regard.

My conclusion is that a cluster of two Orange Pi Ones does not have the resources available to provide stable MapReduce performance. It seemed that as long as the container was successfully created by the ApplicationMaster, the job had a good chance of completing. But many map or reduce tasks just consumed more resources than a single node had available to give while still being responsive, crashing the job.

With detailed analysis and tweaking of the YARN, MapReduce, and Operating system configuration, I believe that small jobs could run stably on my cluster. But the small pool of resources can only be stretched so thin before a node crashes and takes the cluster down with it.

#### Conclusion
Adding more nodes (and therefore more memory) would be the best and most direct approach to stabilizing the performance of my Orange Pi One cluster. However, if directly increasing the cluster's memory resources is not possible, optimization of Hadoop's usage of existing resources is the best strategy. [Hadoop documentation](https://hadoop.apache.org/docs/r2.7.3/), here I come!
