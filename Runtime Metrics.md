#Runtime Metrics (实例度量)

linux容器依赖[cgroup](https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt)，不仅用来跟踪进程组， 并且公布进程组所使用的cpu，内存和块设备IO使用情况。你也能访问这些资源指标，获得网络资源的使用情况。lxc container和docker container在资源限制和配额上使用非常类似。

##Control Groups
Control Groups 是以伪文件系统的形式创建。在最近的linux发行版，可以在`/sys/fs/cgroup`下面发现这个文件系统。在这个目录下， 你能看到多个被称子目录，分别是devices，freezer, blkio等。每个子目录（也叫子系统）对应不同的cgroup层。

在老的系统中，cgroup挂载在`cgroup`,不存在分层。那么，你没法看到多个子目录，你可以看到一群文件和已经创建的container对应的目录。

可以用下面的方式检测cgroup挂载的位置。

<pre>
$grep cgroup /proc/mounts
</pre>

##列举cgroups (Enumerating Cgroups)
进入`/proc/cgroups`,你能看到不同的control cgroup子目录，以及他们所在的分层，和他们包含哪些分层。

你也能在`/proc/<pid>/cgroup`看到进程属于哪个进程组。这些组是层所在的挂载点的根路径下的一个相对路径。例如，`/`意味着进程没有被分在任何一个组，而`/lxc/pumpkin`意味着这些进程是container`pumpkin`的成员。

##寻找一个container的cgroup
对于每个container，一个cgroup会被在每个层中创建。在带有老版本的lxc用户空间工具的旧系统中，cgroup的名字就是container的名字。但是最近的lxc工具里面，cgroup名字变成了`lxc/<container_name>`.

docker container使用cgroups。container的名字就是它的全ID。如果一个`docker ps`发现一个container的名字是ae836c95b4c3 ，通过`docker inspect`或者`docker ps --no-trunc`可以看到他的长ID可能是ae836c95b4c3c9e9179e0e91015512da89fdec91612f63cebae57df9a5444c79（这里只是举例）。

如果要查看container的全部内存信息可以在`/sys/fs/cgroup/memory/lxc/<longid>/`看到。

## cgroups的度量： CPU ,mem, block I/O
对于每个子系统（memory, CPU, and block I/O）你能找到一个或者多个包含统计数据的伪文件

### 内存度量：`memory.stat` 
内存所在的子系统可以看到内存的一些度量数据。内存控制组添加了一些内存使用的强限制（oom之后就container就背kill），因为它能根据主机的内存使用情况做到细粒度的限制。因此，许多linux发行版中选择默认开启这个强限制。一般来说开启之后，你也只需要添加一些linux命令行参数`cgroup_enable=memory swapaccount=1`。

在`memory.stat`里面，你能看到如下统计数据
<pre>
cache 11492564992
rss 1930993664
mapped_file 306728960
pgpgin 406632648
pgpgout 403355412
swap 0
pgfault 728281223
pgmajfault 1724
inactive_anon 46608384
active_anon 1884520448
inactive_file 7003344896
active_file 4489052160
unevictable 32768
hierarchical_memory_limit 9223372036854775807
hierarchical_memsw_limit 9223372036854775807
total_cache 11492564992
total_rss 1930993664
total_mapped_file 306728960
total_pgpgin 406632648
total_pgpgout 403355412
total_swap 0
total_pgfault 728281223
total_pgmajfault 1724
total_inactive_anon 46608384
total_active_anon 1884520448
total_inactive_file 7003344896
total_active_file 4489052160
total_unevictable 32768
</pre>

前半部分是（不带`total_`前缀）包含跟cgroup里面的进程相关的统计数据，但是不包含子cgroup的资源使用情况。带`total_`前缀的包括所有的进程的统计数据。

一些指标像是“仪表”，可以增加也可以减少，例如cgroup使用的交换内存，另外一些像是“计数器”，只增不减。例如pgfault，每次发生缺页都会增加一次。

* cache （缓存内存）
cgroup的进程的cache memory消耗情况跟块设备上的块紧密相关。当你从一个块文件读取并到另一个块文件写入的时候，cache就会增加，使用'conventional' I/O或者mapfile都是同样的场景。即使tmpfs也会出现cache数量上的变化，这个原因尚未查明。

* rss  （进程实际占用内存数，包含共享库占用的内存）
rss包含堆，栈和匿名内存映射等不跟磁盘映射的内存

* mapped_file
cgroup进程的内存映射使用情况。  它并不是告诉你cgroup使用的内存大小，而是告诉你内存是怎么被使用的。

* pgfault and pgmajfault
记录cgroup的进程发生的缺页异常次数以及直接的磁盘访问次数。如果一个进程访问一个不存在或者不被保护的虚拟内存空间就引发一个缺页异常。进程出现bug，尝试访问非法地址（此时程序就会抛一个SIGSEGV信号，发生段错误，杀死进程）。未受保护的虚拟内存是指当进程读到一块被交换出去的或者对应着高内存页的需要映射到实际内存地址的内存，这种情况喜爱内核会从磁盘上加载页，并且让cpu完成内存访问。缺页也发生在进程对写时复制的内存区域写入时。同样的，内核将抢占该进程，复制内存页，唤醒进程在自己独享的内存页上写操作。Major fault是在内核从磁盘读数据的时候发生的。当进程复制已经存在内存的页的时候或者分配一个空的页的时候，这是regular fault。

* swap
cgroup使用的交换内存大小

* active_anon and inactive_anon 
匿名内存被内核标记为活跃和不活跃两种。匿名内存是指那些在文件系统里没有相对应的“储备文件”的那些内存（栈，堆等）。换句话说，它等价于rss counter。实际上rss counter的定义为 active_anon + inactive_anon - tmpfs。active和inactive有什么区别呢？内存页一开始就被标记为active，每隔一段时间，内核扫描内存，并且标记一些页为inactive。不管什么时候被再次访问，它又立即被标记为active。当内核将近耗尽内存的时候，或者需要将内存交换到磁盘，内核会交换inactive页

* active_file and inactive_file
cache内存也有类似于active_anon和inactive_anond的概念。精确的计算方式是 cache=active_file +inactive_file +tmpfs。内核将内存页在active和inactive之间移动的规则跟匿名没存不同，但是基本规则是一样的。注意，当内核需要回收内存的时候，直接从这个inactive池直接取就行，因此可以立即完成，代价很小。但是在匿名内存管理中，匿名页和脏页必须首先写到磁盘。

* unevictable
在这个里面的内存无法被回收。它记录被`mlock`锁住的内存。这个经常在加密框架中用来保存密钥，以免于敏感数据被交换到磁盘。

* memory and memsw limits
这并不是真正的指标，但是是对资源限制的一种提示。前面的用来表示cgroup可用的最大的物理内存。后面的表示ram + swap的最大值。

page cache的计算非常复杂，如果2个进程在不同的cgroup，但是2个进程读了同一个文件，内存交换将被分开，这个意味着一个cgroup终止的时候，它能引起另外一个cgroup的内存使用增加。因为它们没有将其所占的内存页隔离出来独自释放。

##cpu metrics : cpuacct.stat

你已经了解了内存各项指标。相比之下cpu的指标略显简单，全部都能在cpuacct控制器找到。

对于每个container，你都可以找到一个伪文件`cpuacct.stat`,它包含进程的在用户模式和内核模式下的cpu的累计使用情况。如果你对linux比较熟悉，用户模式就是cpu直接控制用户进程的执行（例如执行程序代码），内核模式就是cpu代替进程执行系统调用。

时间片是以1/100秒为一个时钟节拍。
内核定义了USER_HZ来代表用户空间看到的HZ值，在x86体系结构上，USER_HZ值就定义为100。以前这个值跟每秒的节拍数精确映射。但是随着高频调度和[tickless kernels](http://lwn.net/Articles/549580/)的到来，内核的节拍数已经跟时间片没有什么关系了。

##[Block I/O 指标](https://www.kernel.org/doc/Documentation/cgroups/blkio-controller.txt)

这些指标可以在`blkio`控制器看到。不同的指标分布在不同的文件。你能在[blkio-controller](https://www.kernel.org/doc/Documentation/cgroups/blkio-controller.txt)内核文档中找到更加深度的细节。下面只是简介。

* blkio.sectors
包含cgroup进程进行传输的512字节大小的扇区个数。读和写的次数合并在一个计数器。

* blkio.io_service_bytes
cgroup读写的字节数。每个设备有四个计数器。对每个设备来说，都有同步，异步，读和写四种不同的类型。

* blkio.io_serviced
I/O操作数，如上也有四个值。

* blkio.io_queued
IO操作的当前的缓存队列。如果一个cgroup不做任何I/O操作，这些都是0。相反的情况，如果没有IO队列，不意味着cgroup是空闲的，它也许单纯的在quiescent device同步读，这种可以被立即处理，因此不需要缓存队列。但是他有助于描叙cgroup在I/O子系统中压力大小。记住这个只是相对大小。即使一个cgroup的进程不是用很多I/O操作，但是它的队列长度也能随着其他设备负载的上升而增加。

##Network Metrics