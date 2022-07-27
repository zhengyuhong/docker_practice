# 3.3 UTS Namespace

## hostname

使用`docker continaer`进入Docker容器内部发现容器`hostname`已重置并与宿主机隔离

```
docker container run -it ubuntu:20.04 /bin/bash
root@3ecf169ee830:/# hostname
3ecf169ee830
```

容器启动命令添加`--hostname`可指定`hostname`

```
docker container run -it --hostname ubuntu_container_hostname --name ubuntu ubuntu:20.04 /bin/bash
root@ubuntu_container_hostname:/# hostname
ubuntu_container_hostname
```

## namespace

UTS Namespace 主要是用来隔离主机名和域名，本文以 hostname 为例进行 Linux UTS namespace 的介绍。它允许每个 UTS Namespace 拥有一个独立的主机名， 使其在网络上可以被视作一个独立的节点而非主机上的一个进程。例如宿主机的主机名称为`MyLinux`，使用 UTS Namespace 可以实现在容器内的主机名称为 `MyContainerLinux` 或者其他任意自定义主机名。

继续使用第2.2节的联合挂载系统在进行验证下 UTS Namespace 的作用

1. 首先新建一个mount namespace用于挂载新根文件系统

```
unshare --mount --fork /bin/bash # 新建一个命名空间
mkdir -p /root/ubuntu/fs-go/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/fs-go/ /root/ubuntu/fs-go/put_old # 切换根文件系统
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
cd /
```

2. 然后新建一个嵌套uts namespace用于隔离主机名、域名

```
unshare --uts --fork /bin/bash
hostname -b MyContainerLinux
```

3. 最后新建一个嵌套pid namespace隔离进程空间

```
unshare --pid --fork /bin/bash
mount -t proc proc /proc
hostname
```

