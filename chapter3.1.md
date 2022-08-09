# 3.1 Mount Namespace

## namespace

进入第2.2节提及的`/root/ubuntu/rootfs-go/merged`联合挂载点，切换根文件系统

```
cd /root/ubuntu/rootfs-go/merged
chroot . sh
mount # 打印输出只与当前根文件系统相关的挂载目录，不继承复制旧根文件系统挂载目录
```

在 `/mnt` 目录下创建一个目录。使用 mount 命令挂载一个 tmpfs 类型的目录。

```
mkdir -p /mnt/tmpfs
mount -t tmpfs -o size=20m tmpfs /mnt/tmpfs
```

执行完上述命令后，另外新打开一个命令行窗口上查看已挂载的目录信息，可见新根文件系统与旧根文件系统的挂载信息并没有完全隔离，新根文件系统的挂载操作影响旧根文件系统的挂载目录。

```
mount | grep "/mnt/tmpfs"
tmpfs on /root/ubuntu/rootfs-go/merged/mnt/tmpfs type tmpfs (rw,relatime,size=20480k,inode64)
umount /mnt/tmpfs
```

**Mount Namespace** 是 Linux 内核实现的第一个 Namespace，从内核的 2.4.19 版本开始加入。它可以用来隔离不同的进程或进程组看到的挂载点。可以实现在不同的进程中看到不同的挂载目录。使用 Mount Namespace 可以实现容器内只能看到自己的挂载信息，在容器内的挂载操作不会影响主机的挂载目录。

使用以下命令创建一个 bash 进程并且新建一个 Mount Namespace

```shell
unshare --mount --fork /bin/bash # fork一个新的进程，创建一个mount命名空间，执行/bin/bash
```

>执行unshare的进程fork一个新的子进程，在子进程里执行unshare --mount，最后执行/bin/bash
>
>加不加 `--fork` 区别如下：
>
>```
>root@HomeDeb:~# unshare -u bash
>root@HomeDeb:~# ps --forest
>  PID TTY          TIME CMD
> 8134 pts/10   00:00:00 bash
> 9560 pts/10   00:00:00  \_ bash
> 9562 pts/10   00:00:00      \_ ps
>root@HomeDeb:~# exit
>exit
>root@HomeDeb:~# unshare -u -f bash
>root@HomeDeb:~# ps --forest
>  PID TTY          TIME CMD
> 8134 pts/10   00:00:00 bash
> 9563 pts/10   00:00:00  \_ unshare
> 9564 pts/10   00:00:00      \_ bash
> 9566 pts/10   00:00:00          \_ ps
>```
>
>参考 https://zhuanlan.zhihu.com/p/369510683

执行完上述命令后，这时我们已经在主机上创建了一个新的 Mount Namespace，并且当前命令行窗口加入了新创建的 Mount Namespace。下面我通过一个例子来验证下，在独立的 Mount Namespace 内创建挂载目录是不影响主机的挂载目录。

在 `/mnt` 下创建一个目录，使用 mount 命令挂载一个 tmpfs 类型的目录。

```
mkdir -p /mnt/tmpfs
mount -t tmpfs -o size=40m tmpfs /mnt/tmpfs
```

使用`mount`命令查看一下已经挂载的目录信息：

```
mount | grep "/mnt/tmpfs"
tmpfs on /mnt/tmpfs type tmpfs (rw,relatime,size=40960k)
```

可以看到 `/mnt/tmpfs` 目录已经被正确挂载。为了验证主机上并没有挂载此目录，我们新打开一个命令行窗口，同样执行 `mount` 命令查看主机的挂载信息并没有挂载 `/mnt/tmpfs`，可见我们独立的 Mount Namespace 中执行 mount 操作并不会影响主机。

## chroot

从上文可知，使用`chroot`切换根文件系统后，新根文件系统不继承复制旧根文件系统挂载目录，但是在新根文件系统进行挂载会影响旧根文件系统，此时只需要再创建一个Mount Namespace即可隔离新旧文件系统，完整操作如下所示。

```
cd /root/ubuntu/rootfs-go/merged
chroot . sh # 切换新根文件系统
mount # 打印输出只与当前根文件系统相关的挂载目录，不继承复制旧根文件系统挂载目录
unshare --mount --fork /bin/bash # 新建一个 Mount Namespace，与主机挂载目录相互隔离
```

## pivot_root

`unshare --mount --fork /bin/bash` 新建Mount Namespace会继承复制旧Namespace的挂载目录信息，但umount操作并不影响旧根文件系统。

```shell
unshare --mount --fork /bin/bash
mount # 打印输出旧Namespace的挂载目录信息
exit
```

除了 `chroot`，Linux 还提供了 `pivot_root` 系统调用能够将把整个根文件系统切换到一个新的根目录，结合`unshare`、`pivot_root`、`umount -l`实现根文件系统切换和隔离

```shell
unshare --mount --fork /bin/bash # 新建一个挂载命名空间
mkdir -p /root/ubuntu/rootfs-go/merged/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/rootfs-go/merged /root/ubuntu/rootfs-go/merged/put_old # 切换根文件系统
mount -t proc proc /proc
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
mount # 打印输出只与当前根文件系统相关的挂载目录
```

>Linux 提供了一种方式，不直接卸载整个挂载点，而是将挂载点从当前的目录树中脱离（detach），此时所有程序都不再能够通过这个路径来访问该挂载点下的内容，但已经打开的文件描述符等则不受影响。这被称作「懒惰卸载」（lazy unmounting），对应的命令是 `umount -l <path>`。请自行查阅资料（如 [umount(8)](http://man7.org/linux/man-pages/man8/syscall.8.html) 等）。正确隐藏主机的根文件系统后，`/put_old` 应该为一个空目录，此时你可以将它删除（使用 `rmdir /put_old` 即可）

