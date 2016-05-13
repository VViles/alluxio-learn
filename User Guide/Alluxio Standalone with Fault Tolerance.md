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

Alluxio用Zookeeper来实现容错。Alluxio的masters利用Zookeeper进行leader选举。Alluxio的clients也利用Zookeeper获取一致性和获取当前leader的地址。

Zookeeper必须单独安装([ZooKeeper Getting Started](http://zookeeper.apache.org/doc/r3.4.5/zookeeperStarted.html))。

Zookeeper部署好后，注意配置Alluxio的地址和端口。

##Shared Filesystem for Journal

Alluxio需要一个共享的文件系统来存储日志。所有的master必须可以从这个文件系统中读写。但是在任何时间只能有leader master来写入日志，其它的master读取这个日志重置(replay)Alluxio的状态。

这个共享的文件系统必须单独安装，并且在Alluxio服务之前起来。

比如如果用HDFS来存储日志，你必须注意在配置Alluxio时关注NameNode的地址和端口。

##Configuring Alluxio

一旦Zookeeper和共享文件系统运行起来，你必须合理的在每台主机上运行Alluxio-env.sh。

##Externally Visible Address

接下来的几节我们描述外部可见地址。简单的说就是这个地址相当于一个接口，在这个地址上配置好Alluxio后可以被集群的其它节点看见。在EC2上，你应该用ip-x-x-x-x这个地址。特别的，你不能用localhost和127.0.0.1,因为别的节点这样就不能访问你配置的节点了。

##Configuring Fault Tolerant Alluxio

为了开启Alluxio的容错机制，Alluxio masters、workers和clients需要一些额外的配置。在conf/alluxio-env.sh这些选项如下：

|Property Name	|Value			|Meaning
|-----			|------			|
|alluxio.zookeeper.enabled|true	|如果true，masters就会利用Zookeeper开启容错模式|
|alluxio.zookeeper.address|[zookeeper_hostname]:2181|Zookeeper运行的主机名字和端口，用逗号隔开不用的ip地址。|

设置这些选项，你可以配置你知道的ALLUXIO_JAVA_OPTS如下：

    -Dalluxio.zookeeper.enabled=true
    -Dalluxio.zookeeper.address=[zookeeper_hostname]:2181

如果你在使用Zookeeper的集群节点，你需要用逗号隔开不同的地址。比如：

    -Dalluxio.zookeeper.address=[zookeeper_hostname1]:2181,[zookeeper_hostname2]:2181,[zookeeper_hostname3]:2181

或者这些配置可以写在 alluxio-site.properties文件中，更多的细节参考：[ Configuration Settings](http://alluxio.org/documentation/master/en/Configuration-Settings.html)

##Master Configuration

除了上面的配置选项，Alluxio masters还需要额外的配置，接下来的变量必须合理的写在conf/alluxio-env.sh文件中。

    export ALLUXIO_MASTER_HOSTNAME=[externally visible address of this machine]

并且通过设置**ALLUXIO_JAVA_OPTS**中的**alluxio.master.journal.folder**指定具体的日志文件路径。比如如果你用HDFS文件系统，你可以这么添加：

    -Dalluxio.master.journal.folder=hdfs://[namenodeserver]:[namenodeport]/path/to/alluxio/journal

一旦所有的Alluxio masters按照这个方式配置好了，那么Alluxio就具有了容错功能，masters中的一个会成为leader，其它的masters会重置日志文件直到当前的Alluxio master宕机。

##Worker Configuration

只要上述的参数配置正确，worker就会查询Zookeeper，获取当前的leader用来连接。因此**ALLUXIO_MASTER_HOSTNAME**不用为worker配置。

##Client Configuration

只要客户端应用合理的配置如下的参数，那么就不需要额外的配置参数来配成容错模式，客户端应用汇咨询Zookeeper获取当前的leader master。
    
    -Dalluxio.zookeeper.enabled=true
    -Dalluxio.zookeeper.address=[zookeeper_hostname]:2181

