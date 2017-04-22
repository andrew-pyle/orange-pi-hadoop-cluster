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
