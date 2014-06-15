#container的安全特性

　　审视docker的安全性应该从下面三个主要的因素考虑：
　 

* 内核本身支持的命名空间和cgroup的安全性
* docker守护进程暴露出来的[攻击点](http://en.wikipedia.org/wiki/Attack_surface)
* 内核和container交互的安全底线(the "hardening" security features)
    
##内核命名空间
　　docker的container本质上是一个lxc container，因此他们在安全特性上是相同的。当你用`docker run`起一个container的时候，背后其实就是
`lxc-start`在操作这个container。`lxc-start`会创建一系列的资源隔离和限制，而不是docker本身做。因此docker可以很轻易的使用lxc提供的如
`LXC userland tools evolve｀等特点

　　命名空间提供最初级和最直接的资源隔离: 一个container内部的进程无法看到或者说尽可能少的影响到其他的container，甚至是宿主机本身。

　　每个container应该有自己的网络协议栈(network stack)，而不会有其他container的套接字(sockets)或者接口(interface)的使用权限。当然如果一个主机系统做一些相关配置的话，container也能通过各自的网络接口跟主机上其他的container交互。当你对外公开特定端口，或者使用[`link`](https://docs.docker.com/userguide/dockerlinks/#working-with-links-names),container之间就能互相ping通，收发udp包，建立tcp链接等。在网络架构设计上，docker在一个主机上启动的container是通过桥接的方式跟外部通信的，docker提供一个网桥,类似与物理机上的以太网交换器(Ethernet switch),container直接通过网桥发送和接受数据。

　　那么，实现命名空间和私有网络的代码有多成熟呢？内核命名空间在[`between kernel version 2.6.15 and 2.6.26`](http://lxc.sourceforge.net/index.php/about/kernel-namespaces/)首次被引入，因此自从２００８年６月份以来，namespace的代码就会经受大量的生产环境的训练和压测,并且，其namespace的设计和灵感比这个更老,namespace正在努力的重新实现OpenVZ来让他支持嵌入到主流操作系统，OpenVZ首次在２００５年推出发行版。因此设计和实现都是相当成熟了。

##资源限制
　　control group(cgroup)是linux　container的另外一个关键组件。他主要实现资源的配额和限制。他提供了许多有用的指标，并且保证每个container获得他们该有的cpu,memory,disk I/O等。更重要的是，它能保证单个container的资源耗尽不会导致整个系统的崩溃。
　　虽然cgroup在限制一个实例访问和修改另外一个实例的数据和进程的无法发挥作用，但是可以有效的抵挡拒绝服务攻击(denial-of-service attacks)。特别是在例如私有和公有PAAS等多租户平台，cgroup能在一些应用出现非正常操作的时候保证机器一致的正常运行时间(和性能)。
　　cgroup曾经有一段时间停滞:其代码２００６年开始编写，直到kernel 2.6.24才被合并进去。

##docker守护进程的攻击面
　　在docker中运行container之前必须要运行docker守护进程。其必须有root权限才能启动，因此必须注意一些重要的细节。
　　首先，只有可信任的用户才能被授权去控制docker守护进程。一些很强大的特性也是因此而生。特别地，docker允许一个container和宿主机共享文件夹，并且允许container毫无限制的访问权限。这意味着你能启动一个container，在container里面/host目录就是主机的根目录，container也能无限制的修改你文件系统的结构。不可思议吧？但是所有的虚拟系统都允许在共享文件上面干类似的事情。在虚拟机上共享根分区(根分区的块设备)之后上面事情都可能发生。
　　这里有一个很大的安全隐患：如果你在一个web服务器上搭建docker服务，使用API的方式来对外提供container，你就应该对所有的入参做严格的检查，预防恶意用户通过"精心准备"的参数来随心所欲的创建container。
　　因此，REST API endpoint在0.5.2版本之后发生了变化。如果在本地直接运行docker，你就可以使用unix sock而不是tcp socket来控制docker，以防止类似跨站攻击(cross-site-scripting)等。unix sock的权限控制可以使用传统的unix权限检查来保证。
　　在http的基础上，你可以选择对外开放REST API。但是，你应该意识到上面可能的安全隐患,确保通过可信任的网络或者VPN进来的请求才被处理，或者使用stunnel或者对客户端进行SSL证书的验证。
　linux namespace的最近实现版本实现了user namespace,可以让docker在非root权限下创建拥有所有特性的container。[点此](http://s3hh.wordpress.com/2013/07/19/creating-and-using-containers-without-privilege/)查看详情。将container内部的用户对应到主机上的用户，还能能解决主机和客户机之间的文件共享的问题。
　　docker最终目标是实现２个方面的安全提升：
* 将container的root用户对应到宿主机上的非root用户，减少container-to-host的权限升级带来的影响
* 允许docker守护进程在非root权限下面运行，仅仅委派一些操作权限给严格验证过的子进程组，并且保证子进程资源的限制范围，例如虚拟网络创建，文件系统管理等。
　　最后，如果在服务器上运行docker，就让服务器上其他的服务尽量的跑在docker内部。当然除了一些命令行工具和一些监控和监护进程。
##linux内核Capabilities
　　默认情况下，docker是在非常严格的进程权能集合下启动container。这代表什么呢？
　　进程权能能将目前的root/non-root的权限管理方式分割成更加细粒度的访问控制系统。进程号小于1024的进程不再需要root启动，而是被赋予``net_bind_service`。还有许多其他的权能，几乎可以覆盖所有需要root权限的区域。
　　这对安全有着重要的意义。我们来看看为什么。
　　一个典型的服务器需要root启动一批进程，包括SSH,cron,syslogd,硬件管理工具，网络配置工具等，container是非常不同，因为几乎所有的任务都是在容器内的设备上被处理的。
* SSH的访问控制应该是在container内部的一个服务来管理
* cron应该跑在为app量身定制的用户进程里面，而不是平台维度的基础设施。
* 日志管理应该交给docker，或者像Loggly等第三方服务来处理
* 硬件管理是无关的，你不用在container里面执行`udevd`或者同等作用的守护进程
* 网络管理应该在container外部尽可能的隔离，因此在container内部也不需要要ifconfig,route和ip命令。

　　那么在大部分情况下面，container不需要root权限。只需要在一个简化的进程权能集合，因此container内部的root比实际的root权限要小很多。比如，以下都是可以实现的：
* 禁止所有的mount操作
* 禁止访问raw sockets（可以防止包欺骗）
* 禁止某些文件系统的操作，比如创建新的设备文件，改变文件的owner或者修改某些属性。
* 禁止加载模块
* 更多其他

　　因此即使一个恶意用户获得了root权限，也很难对宿主机做出严重的破坏行为。

　　这不影响正常的web应用。但是恶意用户的破坏力大大消减。[这里](https://github.com/dotcloud/docker/blob/v0.5.0/lxc_template.go#L97)是在docker里面被移除的权能清单,[这里](http://man7.org/linux/man-pages/man7/capabilities.7.html)是所有的权能清单。当然，如果你需要额外增加一个必要的权能，你也可以合理配置。docker container默认关闭额外的权能来保证最大的安全性。
　　
## 一些其他的内核安全属性
　　权能是现代操作系统安全特性中的一种。现有的例如TOMOYO, AppArmor, SELinux, GRSEC等都可以用在docker里面。
　　当前docker激活了权能，但是并不妨碍它使用其他的一些系统来加固docker主机的安全。例如：
* 在GRSEC and PAX下面运行内核，它会在程序的编译和执行阶段进行很多安全相关的检查，它比地址随机化技术更加有用。它不需要docker进行额外配置，跟container没有依赖关系。
* 如果你使用lxc container自带的安全模型的模板，你也能在沙盒外面使用。例如，Ubuntu自带的AppArmer模板。这些模板提供额外的安全。
* 你可以使用你最喜欢的访问控制机制来定义你的安全策略。docker container就是实打实的lxc container。

　　有很多第三方的工具，例如网络拓扑或者共享文件。通过这些工具可以在不影响已有的核心功能的前提下来加固docker container。
　　

## 总结
　　docker container是非常安全的。特别是你用非root权限在container内部运行服务的时候。
　　也可以通过激活额外的安全控制层，例如Apparmor, SELinux, GRSEC或者任何你喜欢的安全解决方案。
　　同样重要的，在其他你感兴趣的容器类系统的安全特性，都可以在docker里面实现。毕竟这些都是在内核级别实现的。
　　
