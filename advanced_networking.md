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

* `--ip=IP_ADDRESS` — see Binding container ports

* `--ip-forward=true|false` — see Communication between containers

* `--iptables=true|false` — see Communication between containers

* `--mtu=BYTES` — see Customizing docker0
　　
　　有２个配置在docker守护进程或者`docker run`的时候可以修改。
* `--dns=IP_ADDRESS`
* `--dns-search=DOMAIN...`

　　一些网络配置只能在`docker run`的时候指定，它能自定义很多特殊的选项。
* `-h HOSTNAME` or `--hostname=HOSTNAME` -- see  Configuring DNS and How Docker networks a container
* `--link=CONTAINER_NAME:ALIAS`  — see Configuring DNS and Communication between containers
* `--net=bridge|none|host|container:NAME_or_ID`  — see How Docker networks a container
* `-p SPEC` or `--public=spec`  --see Binding container ports
* `-P` or `--publish-all=true|false` --see Binding container ports

　　下面由浅入深的介绍上面的这些话题。

## DNS配置(Configuring DNS)
　　如果不将主机名在镜像编译的时候定义好，docker如何给每个container生成一个主机名和dns配置？有个很机智的方法，通过覆盖container下面三个`/etc`文件，这三个文件可以写入。在container内部运行`mount`你能看到这三个文件:
<pre>
$$ mount
...
/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...
tmpfs on /etc/resolv.conf type tmpfs ...
...
</pre>
　　这种管理允许docker可以做一些很灵活的事情。在DHCP自动修改网络配置自后，这样可以保持所有constainer的resov.conf是最新的。这些配置文件不随着docker版本的改变而改变。所以你可以用下面的参数来重新定义。
　　有四个影响域名服务的因素：
* `-h HOSTNAME` or `--hostname=HOSTNAME` ——设置主机名，直接写进`/etc/hostname`和`/etc/hosts`,作为ip地址对应的主机名，并且在`/bin/bash`的预定义变量里面。但是这个主机名在container的外部不可见，你也不能在docker ps或者在其他container的`/etc/hosts`设置里面出现生效。
* `--link=CONTAINER_NAME:ALIAS` ——创建container的时候指定，它在`/etc/hosts`下面新增一个一对ip和主机名,ip是CONTAINER_NAME对应的ip,主机名是ALIAS。这就可以让新建的container和--link指定的container相互通信，直接使用ALIAS而不需要IP。

* `--dns=IP_ADDRESS` ——设置`IP_ADDRESS`作于域名解析服务器，添加在`/etc/resolv.conf`,在container内部的进程如果遇到一个主机名不在`/etc/hosts`下面，就会直接去请求这个地址和其53端口的域名解析服务。

* `--dns-search=DOMAIN...`——设置`/etc/resolv.conf/`的`dns-search`，当一个进程的试图访问`host`，并且dns-search设置了`example.com`，DNS不仅仅查找`host`，而且也会查找`host.example.com`。

　　主要上面的后两项不设置，那么`/etc/resolv.conf`就会复制docker守护进程所在主机的`/etc/resolv.conf`一份。设置的选项会在此基础上修改。

##container之间的通信
　　在操作系统级别，有三种方式管理container之间的通信。
1, 现有的网络拓扑结构可以链接每个容器的接口么？默认，所有的container都链接在同一块网桥`docker0`下面，并由此来转发数据包。下面章节会介绍其他的拓扑结构
2, 宿主机会支持IP转发么？设置系统参数`ip_forawrd`可以支持。只有被设置为1的时候，container之间才能互相转发。默认情况下，docker守护进程在启动的时候会将其设置为１。可以如下验证，人工设置：
<pre>
# Usually not necessary: turning on forwarding,
# on the host where your Docker server is running

$ cat /proc/sys/net/ipv4/ip_forward
0
$ sudo echo 1 > /proc/sys/net/ipv4/ip_forward
$ cat /proc/sys/net/ipv4/ip_forward
1
</pre>
3, 支持iptables去进行一些特殊的连接么？如果你在启动docker守护进程的时候已经设置`--iptables=false`,docker就不会动你的iptables规则。如果设置`--icc=true`，docker会增加一条默认的转发链(FORWARD)，并且设置为"ACCEPT"。否则，如果`--icc=false`，转发规则会将其设置"DROP"。

　　几乎每个人都想打开`ip_forward`选项，保证container之间的通信。但是难以决策的是`--icc`的设置。所以，iptables就能保护container，包括宿主在内，防止他们的安全性不高的端口被探测或者被请求。
　　`--icc=false`是最安全的模式。但是这种情况下，container之间如何通信呢？
　　答案是通过`--link=CONTAINER_NAME:ALIAS`，如果设置了`--iptables=true`，docker就会在对应的２个container之间设置一对`iptable ACCEPT`规则，通过在dockerfile中`EXPOSE`提到的的端口，一个container就能访问另外一个。
　　通过`iptables`命令，在宿主机器上就可以看到`FORWARD`链的策略是`ACCEPT`还是`DROP`。
<pre>
sudo iptable -L -n...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
...

# When a --link= has been created under --icc=false,
# you should see port-specific ACCEPT rules overriding
# the subsequent DROP policy for all other packets:

$ sudo iptables -L -n
...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  172.17.0.2           172.17.0.3           tcp spt:80
ACCEPT     tcp  --  172.17.0.3           172.17.0.2           tcp dpt:80
DROP       all  --  0.0.0.0/0            0.0.0.0/0
</pre>
注意，主机唯独的iptables规则完全将container的原ip地址暴露出来了，因此从一个container向另外一个container的连接看起来就像是从container本身的ip地址发出。

#绑定container的端口到主机
　　默认，docker container可以连接到外网，但是外面无法连接container。通过`iptables`地址伪装(iptables masquerade，动态获得ip的SNAT)，使得每个通向外部的连接就像是宿主机本身发起的。转发规则在docker守护进程启动的时候指定:
<pre>
# You can see that the Docker server creates a
# masquerade rule that let containers connect
# to IP addresses in the outside world:

$ sudo iptables -t nat -L -n
...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16       !172.17.0.0/16
</pre>
　　如果你想要container接受外部的请求，需要在`docker run`的时候提供更多的选项。
　　首先，`docker run`时指定`-P`或者`--publish-all=true|false`，镜像文件的dockerfile中`EXPOSE`选项一个一个指定的端口被映射到主机上49000-49900这个范围。这有点不方便的是，你还需要通过`docker port`来获取container的一个端口究竟被映射到外部的哪一个。
　　相对方便的是，`-p SPEC`和`--publish=SPEC`可以让你显式的指定container内部的端口到宿主端口的映射，并且宿主的端口不一定是在49000-49900之间。
　　不管哪种方法，查看iptables的nat表，都能看到他们的转发信息。
<pre>
# What your NAT rules might look like when Docker
# is finished setting up a -P forward:

$ iptables -t nat -L -n
...
Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:49153 to:172.17.0.2:80

# What your NAT rules might look like when Docker
# is finished setting up a -p 80:80 forward:

Chain DOCKER (2 references)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.17.0.2:80
</pre>
　　由上面可以看到，docker已经在`0.0.0.0`这个跟通配ip地址上开放一些container的端口。如果你像更加严格，只想让container访问宿主机某个接口，执行`docker run`的时候使用`-p IP:host_port:container_port`或者`-p IP::port`来绑定宿主机器上指定的接口。
　　如果你经常docker端口转发绑定到一个指定ip地址，你可以在修改一些系统级别的docker守护进程的参数(在ubuntu上，编辑`/etc/default/docker`的`DOCKER_OPTS`选项)，增加一个`--ip=IP_ADDRESS`选项，并且重启docker守护进程。

## 自定义网桥docker0
　　默认，docker server在主机上创建并且配置一个接口`docker0`,作为一个内置内核的以太网网桥，它可以在虚拟或者物理的网络接口之间转发数据包，组成一个单独的以太网。
　　docker启动的时候，配置`docker0`的ip地址和掩码使得宿主机可以通过它来跟container通信，配置MTU来限制接口上传输的最大包大小。
* `--bip=CIDR` ——指定网桥的IP地址和子网掩码，使用标准的CIDR(无类别域间路由)标记,例如`192.168.1.5/24`。
* `--mtu=BYTES` 
    在ubuntu上，可以添加这些选项到`/etc/default/docker.io`。

　　`brctl`命令可以帮你看到你的container是否正确的连接到`docker0`。在宿主机器上执行`brctl`，在`interfaces`这一列可以看到网桥上的连接的接口。
   下面是一个有２个container的机器。
<pre>
# Display bridge info

$ sudo brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.3a1d7362b4ee       no              veth65f9
                                                        vethdda6
</pre>　

    在docket主机上如果没有brew命令，并且你的系统是ubuntu的，`sudo apt-get install bridge-utils`来安装。
　　最后，`docker0`在每次创建container的时候都，在网桥构成的子网里面选择一个可用的ip，在端口eth0上设置对应的ip和掩码。这个ip也是container访问外网的网关。
<pre>
# The network, as seen from a container

$ sudo docker run -i -t --rm base /bin/bash

$$ ip addr show eth0
24: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 32:6f:e0:35:57:91 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::306f:e0ff:fe35:5791/64 scope link
       valid_lft forever preferred_lft forever

$$ ip route
default via 172.17.42.1 dev eth0
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.0.3

$$ exit
</pre>
记住，设置宿主机的`ip_forward`为１才能转发container的数据包。

## 绑定container到你自己的网桥上
　　如果你不需要docker给你创建网桥，那么通过`-b BRIDGE` 或者 `--bridge=BRIDGE`指定你的网桥。已经运行的使用网桥docker0的docker server可以通过下面的方式来停止服务并且移除接口docker0.
<pre>
# Stopping Docker and removing docker0

$ sudo service docker stop
$ sudo ip link set dev docker0 down
$ sudo brctl delbr docker0
</pre>
　　在启动docker server之前，创建你的网桥，并且配置。下面我们创建一个足够简单的网桥，就像我们先前自定义的`docker0`。但是足够说明这个技巧。
<pre>
# Create our own bridge

$ sudo brctl addbr bridge0
$ sudo ip addr add 192.168.5.1/24 dev bridge0
$ sudo ip link set dev bridge0 up

# Confirming that our bridge is up and running

$ ip addr show bridge0
4: bridge0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state UP group default
    link/ether 66:38:d0:0d:76:18 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.1/24 scope global bridge0
       valid_lft forever preferred_lft forever

# Tell Docker about it and restart (on Ubuntu)

$ echo 'DOCKER_OPTS="-b=bridge0"' >> /etc/default/docker
$ sudo service docker start
</pre>
　　启动docker，并且创建一个container,你将看到container的ip是在你指定的ip段内的。

## 配置一个container的网络
　　docker目前处于活跃的开发之中，会不断的调整网络配置逻辑。这一节通过shell命令大致的描叙了docker新建container的过程中网络配置的步骤。
　　先回顾一些基本的概念。
　　使用IP通信，一个主机至少需要要连接上一个可以转发数据包的网络接口，一个定义接口可到达的IP地址段的路由表。网络接口不一定是物理设备。实际上,`lo`环回接口就是虚拟的，它仅仅从内存中拷贝发送者的数据到接受者。
　　docker使用特别的虚拟接口使得container可以跟主机通信——一对叫做`peers`的虚拟接口连接在主机的内核，数据包通过它传输。下面就是docker配置网络的简单步骤：
1, 创建一对虚拟接口
2, 给其中一端，取唯一名，如`veth65f9`，绑定到`docker0`或者用户自定义的网桥上。
3, Toss the other interface over the wall into the new container (which will already have been provided with an lo interface) and rename it to the much prettier name eth0 since, inside of the container’s separate and unique network interface namespace, there are no physical interfaces with which this name could collide.

4, Give the container’s eth0 a new IP address from within the bridge’s range of network addresses, and set its default route to the IP address that the Docker host owns on the bridge.
　　当然你也可以不使用上面的步骤来创建。在docker run的时候配置`--net=`,它有四个值。
* `--net=bridge` ——默认值
* `--net=host` ——不设置单独的网络栈。本质上，这个选项是告诉docker不要去容器化container的网络配置。`ip addr`可以发现，container可以直接访问宿主机所在的网络接口。`--privileged=true`设置后，container就可以对网络栈进行重新配置。但是container进程也不能像其他root进程一样打开低位的端口。
* `--net=container:NAME_or_ID` ——告知docker将进程创建在其他的container的网络栈里面。两个container共享ip地址和端口。通过环回地址也能相互通信。
* `--net=none` ——将container的网络创建在docker所在的网络栈，但是不进行任何的配置。让用户完全自定义。
　　如果你使用`--net=none`,下面这些步骤可以的帮你创建跟docker默认基本一样的网络配置。

<pre>

#At one shell, start a container and
#leave its shell idle and running

$ sudo docker run -i -t --rm --net=none base /bin/bash
root@63f36fc01b5f:/#
# At another shell, learn the container process ID
# and create its namespace entry in /var/run/netns/
# for the "ip netns" command we will be using below

$ sudo docker inspect -f '{{.State.Pid}}' 63f36fc01b5f
2778
$ pid=2778
$ sudo mkdir -p /var/run/netns
$ sudo ln -s /proc/$pid/ns/net /var/run/netns/$pid

# Check the bridge’s IP address and netmask

$ ip addr show docker0
21: docker0: ...
inet 172.17.42.1/16 scope global docker0
...
# Create a pair of "peer" interfaces A and B,
# bind the A end to the bridge, and bring it up

$ sudo ip link add A type veth peer name B
$ sudo brctl addif docker0 A
$ sudo ip link set A up

# Place B inside the containers network namespace,
# rename to eth0, and activate it with a free IP

$ sudo ip link set B netns $pid
$ sudo ip netns exec $pid ip link set dev B name eth0
$ sudo ip netns exec $pid ip link set eth0 up
$ sudo ip netns exec $pid ip addr add 172.17.42.99/16 dev eth0
$ sudo ip netns exec $pid ip route add default via 172.17.42.1

</pre>
　　

## 创建一个p2p连接
    

