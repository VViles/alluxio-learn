#Alluxio Standalone with Fault Tolerance

----------

Alluxio中的容错机制是通过多master的方式实现的，多master就是在不同的机器上同时起master进程。
然后在这些master中选择出一个leader来作为workers和clients的主要联系者。另外的master利用共享日志成为standby的角色。这样的方式保证他们在leader宕机后可以马上成为新的leader，并且快速接管后维护的是同一份文件系统的元数据。

如果leader宕机后，新的leader会从standby的masters中自动选择出来，并且Alluxio的服务继续运行。注意这里如果替换成standby master后，clients可能会有短暂的延迟或者瞬时的错误。

##Prerequisites


配置Alluxio集群的容错需要两个前提条件。

* [Zookeeper](http://zookeeper.apache.org/)
* 一个共享的可靠的底层文件系统用来放置日志。

Alluxio用Zookeeper的leader selection来实现容错。这样保证了在Alluxio运行的任何时间都有至少有一个master在运行。

Alluxio也需要一个共享的可靠的底层文件系统用来放置日志。这个共享的文件系统必须能被所有的masters访问的到，提供的可能选项包括: HDFS, Amazon S3和GlusterFS。主master写日志到共享的文件系统，然后standby master持续的重置日志条目来保证是最新的。

##ZooKeeper

Alluxio用Zookeeper来实现容错。