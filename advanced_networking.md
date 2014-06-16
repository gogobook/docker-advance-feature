#docker高级网络配置

　　docker守护进程启动之后，就会在宿主机上创建一个虚拟网络接口`docker0`。根据[RFC 1918](http://tools.ietf.org/html/rfc1918),docker随机选择一个主机用不到的专用范围的地址和子网。把它分配给`docker0`。Docker目前在启动的时候选择`172.17.42.1/16`，其中16位网关，65536个主机地址，提供给主机和container使用。
　　但是docker0并不是随意的接口。它是一个虚拟网桥，自动转发其他网络接口发送给它的数据。这可以让container使用它跟宿主机以及他们互相之间进行通信。每次container被创建，同时也创建一对同等的接口，就像pipe的对端一样，在一端发送一个包就能在另一端接收到。这样的一对接口一端在container内部成为`eth0`，另一端的接口在主机的命名空间下面，如`vethAQI2QT`。通过绑定`vth*`接口到`docker０`网桥，docker就在宿主机和container之间创建了一个虚拟的共享子网。
　　接下来的章节提供了很多方法，通过使用docker提供的选项，或者更加高级的原生的[linux网络配置命令](https://blog.kghost.info/2013/03/01/linux-network-emulator/
)，来调整，补充或者替换docker的默认网络配置。

## 配置选项的快速指导
　　有一些选项在启动docker守护进程的时候就确定了，因此启动后就无法被改变：
* `-b BRIGE` or `--brige=BRIGE` --see [Building your own bridge](#bind)
* `-bip=CIDR` --see [Custermizing docker0](#docker0)
* `-H SOCKET ...` or `--host=SOCKET...` 听起来是否可以影响container网络配置，实际上相反，它告诉docker守护进程那一条channel它可以去接受docker客户端的命令.
* `--icc=true|false` see []()
* `--icc=true|false` — see Communication between containers

*`--ip=IP_ADDRESS` — see Binding container ports

*`--ip-forward=true|false` — see Communication between containers

*`--iptables=true|false` — see Communication between containers

*`--mtu=BYTES` — see Customizing docker0
　　
　　有２个配置在docker守护进程或者`docker run`的时候可以修改。
* `--dns=IP_ADDRESS`
* `--dns-search=DOMAIN...`

　　最后一些网络配置只能在`docker run`的时候指定，它能自定义很多特殊的选项。

