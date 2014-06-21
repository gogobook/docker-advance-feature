#docker文件系统

##介绍
[docker-filesystems-generic](!docker-filesystems-generic.png)

为了可以让内核跑起来，通常需要两种[文件系统](http://en.wikipedia.org/wiki/Filesystem).
1 boot文件系统(bootfs,引导文件系统)
2 root文件系统(rootfs，根文件系统)

boot文件系统包括引导程序(bootloader)和内核。用户无法改变bootfs.实际上，引导进程处理完毕之后，整个内核就加载到了内存，先后写在bootfs，initrd RAM磁盘，并释放相对应的内存。

根文件系统通常包括Unix-Like操作系统的文件目录结构：
`/dev, /proc, /bin, /etc, /lib, /usr, /tmp`。包括运行用户应用程序需要的所有的配置文件，二进制文件以及链接库（如ls等）

但是不同的linux发行套件之间内核的区别很大。根文件系统的内容以及组织因发行套件不同而导致你的软件包的不同。 docker通过同时运行多个发行套件可以解决这个问题。

[docker-filesystems-multiroot](!docker-filesystems-multiroot.png)