---
layout: post
title:  一次数据库性能调优实践
subtitle  : IO系统，调优实践，方法，TokuDB
tags : RDBMS, TokuDB
---

> 本文大概分成两个主要部分:
>  * 跳出任何一种数据库，一个以磁盘为主要存储介质的DB，性能度量指标有哪些，那些基本属性的底层机制是什么。
>  * TokuDB performance tuning 以及为了追求最大性能要做出哪些必要的牺牲。

## DB Workload
&emsp;&emsp;吞吐跟延时是两个相对宏观性能度量标准，这些标准在不同系统的表现也有所差别。那么该如何描述DB的负载情况呢？一个负载特征描述大概是这样的。*某内容社区的数据库，平均的读写比例为3：7，读操作频率为1000次/秒，每次读的数据小于1KB，写频率相对读较高，平均写入的数据大小在1M。写操作在业务峰值时能达到8000次/秒* 从中可以提炼出几个最基本的“原子”属性来刻画“宏观”的性能标准，具体如下：
* 文件IO吞吐和延时
* IO尺寸
* 操作的频率和类型

&emsp;&emsp; 在Linux中并没有查看文件系统延时的接口，其提供的性能度量手段是低层次的Disk级别的指标信息。如果不了解底层的话，在performance的时候往往抓不住重点，哪些情况下文件系统的延时与上层应用的性能成正比，哪些又不受影响呢？例如：从指标上看磁盘IO突然处在波峰上，但应用的指标延时却很平稳，那么很可能是文件系统的后台线程在刷磁盘，没有任何上层应用要等待这些操作完成。那么有没有一些本质的东西呢？作为一个工程师，你如果理解了数据如何在分级存储的系统中移动的，那么在自己写程序的时候就可以把数据放在离应用最近的地方以提升性能。实际上无论宏观架构还是微观的调优 *存储分级和就近计算（局部性）* 都是计算机体系中两个最基本的思想，如果应用所需的数据在CPU寄存器中，指令周期<=1，如果在高速缓存中，指令周期大概在7/80左右，如果在主存上，需要几百个指令周期，那么如果数据在磁盘上，需要的CPU指令周期是千万级别的，由此可见数据距离计算越近应用延时越低，性能越好。所以存储是具有层次结构的，其设计的指导思想是，对于每一层，其上层都作为本层的Cache，同时自己又作为比自己低层次的Cache。

## Storage Layer
![](https://raw.githubusercontent.com/pangzhenzhou/pzz-github-pages-blog/gh-pages/public/image/storage-lager.png)

文件系统利用Cache提高读性能，利用Buffer来提高写性能，在Linux系统中几乎所有的文件读写操作都依赖于Page Cache(在较早版本的Kernal中还有Buffer Cache)，但是也有例外，例如：
* Raw IO： 绕过整个文件系统直接把数据发送给磁盘。
* Direct IO： 绕过缓存直接使用文件系统，这种情况下预读这些常规的优化手段可能会失效。
通常来说自建缓存的基础软件——数据库，倾向于以自己的方式来管理内存，这在性能上更优，少写了一部分缓存，减少了内存拷贝。到这可以看出来，文件系统所做的工作不单单是把数据存下来，提供一个read，write 接口那么简单。为了弥合两端处理速度的不平衡，它要管理cache，管理元数据，甚至会发起额外的IO操作。

总结：系统优化两个大方向永远没有错，减少内存Copy，比如文件访问时mmap，Direct IO等。减少重复计算利用好系统的局部性，比如存储分级等，以更为优化的方式使用硬件，比如预读等，同时了解应用的workload也同样重要。这样才能更为针对性的优化。

## Tokudb Performance
&emsp;&emsp;这部分我们使用问答的方式来说一些方法及流程步骤，原理性的东西不过多讲。

1. 为什么选择TokuDB
   * 对于写比读更heavy的业务，TokuDB是更为合适的，TokuDB 使用Fractal Tree模型，避免了B Tree在写的时候由于Page节点分裂，造成磁盘访问方式变得更随机的问题。
   * TokuDB支持事务，与InooDB更为兼容。
   * HotBackup更为简单，TokuDB支持[check point lock](https://github.com/percona/tokudb-engine/wiki/Checkpoint-Lock)，可以在较小影响的线上业务情况下，用直接copy数据文件的方式来备份恢复。这地方是由一些坑的，通常来说为了不出问题建议数据库实例的数据目录保持相同，数据字典及log文件要一起copy。
   * 存储密度更高——TokuDB支持数据压缩，可以在建表的时候指定，带来的副作用就是cpu负载会更高一些，关于压缩：TokuDB 默认的块大小是4M,InnoDB 需要需要将压缩后的数据pad到固定大小的空间里。这多少会影响压缩的效果。
   * 支持在线加索引。
2. 性能测试工具
   * 在物理机交付时，一定要做用Fio做IO系统的性能测试，如果把Fio的各种参数搞明白，对Linux IO 就完全掌握了。可以覆盖到所有的文件访问方式
   ----------
   - sync：Basic read(2) or write(2) I/O. fseek(2) is used to position the I/O location.
   - psync：Basic pread(2) or pwrite(2) I/O.
   - vsync: Basic readv(2) or writev(2) I/O. Will emulate queuing by coalescing adjacents IOs into a single submission.
   - libaio: Linux native asynchronous I/O.
   - posixaio: glibc POSIX asynchronous I/O using aio_read(3) and aio_write(3).
   - mmap: File is memory mapped with mmap(2) and data copied using memcpy(3).
   - splice： splice(2) is used to transfer the data and vmsplice(2) to transfer data from user-space to the kernel.
   - syslet-rw： Use the syslet system calls to make regular read/write asynchronous.
   - sg：SCSI generic sg v3 I/O.
   - net ： Transfer over the network. filename must be set appropriately to ‘host/port’ regardless of data direction. If receiving,only the port argument is used.
   - netsplice： Like net, but uses splice(2) and vmsplice(2) to map data and send/receive.
   guasi The GUASI I/O engine is the Generic Userspace Asynchronous Syscall Interface approach to asycnronous I/O.

   ----------
   * IO调度算法设置为[DeadLine](https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html)，即便是SSD盘也非常有必要，在测试过程中DeadLine比Cfq会有很大提升。可以在调整前后利用Fio做对比测试。
   * [TPCC MySQL](https://github.com/Percona-Lab/tpcc-mysql)

3. [可调参数](https://www.percona.com/doc/percona-server/LATEST/tokudb/tokudb_variables.html)
   * tokudb_directio=ON, 通常来说DirectIO要开，但是如果是一个节点多实例部署，比如每个磁盘部署一个instance，不建议打开。
   * tokudb_cache_size=XXX，TokuDB实例缓存大小，要设置为更为合理的数值，通常来说物理机内存的60%左右，同样多实例部署后Server的over head还是很大的。
   * tokudb_fsync_log_period=XXX，这个参数会大幅提高性能，但是要谨慎是会牺牲一下D的。（事务的ACID）
   * sync_binlog = XXX，不在极端情况下不建议打开，打开这个参数binlog刷磁盘完全是由文件系统来控制的。
   * table_definition_cache = XXX，table_open_cache=XXX，如果数据库中shard了非常多的表建议调大这个值。
   * 在Slave端 RFR 建议打开 tokudb_rpl_unique_checks=0 , tokudb_rpl_lookup_rows=0，打开这个意味着取消了一致性检查。
4. 监控报警HA
   * Percona [PMM](https://www.percona.com/doc/percona-monitoring-and-management/index.html) ，非常赞，不仅支持TokuDB，对InnoDB，MongoDB也是支持的。其底层的指标是存储在Prometheus的，如果公司的运维体系是也是采用同样的方案，整合起来是很方便的，支持RestFul接口。对程序友好。
   * TokuDB 暂时还没有Group Replication，可以采用基于MHA的方案自动切换。具体思路可以是这样：由于MHA进程完全无状态，可以跑在基于k8s的Pass平台上（或者用supervisor），然后利用MHA的hook script 在db topology 节点变化的时候将新的Master写到etcd/Zookeeper上，通过分布式协调系统的Watch功能，基本可以实现对application无感知的，自动切换。
   * cpu的load报警阈值可以适当调高，IO程序通常排队情况比较严重。
   * 保持数据盘的容量<=70%，尤其是HDD盘，磁盘的容量对IO性能影响是有的，一个磁盘的前三分之一跟后三分之一性能也是不同的，当磁盘容量下降，写入新数据的时，一方面需要花更多的时候在磁盘上找空闲的块，而磁盘Seek本身就是个偏随机的操作成本很高，随着磁盘容量的减少，碎片也会加剧，磁盘上的空间就会变得更小更碎了。
