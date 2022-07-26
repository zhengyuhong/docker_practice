# 3.2 PID Namespace

## PID

使用过Docker容器的朋友应该都知道，容器内部的进程和外部进程是相互隔离的，如下启动一个ubuntu容器，执行`ps`，仅包含容器内部的进程，PID=1的为容器启动后第一个进程。

```
docker container run -it ubuntu:20.04 /bin/bash
root@7f1b338325d2:/# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   10 pts/0    00:00:00 ps
```

在Linux系统中，进程PID从1开始往后不断增加，并且不能重复（当然进程退出后，PID会被回收再利用），PID为1的进程是内核启动的第一个应用层进程，一般是init进程，具有特殊意义，当系统中一个进程的父进程退出时，内核会指定init进程成为这个进程的新父进程，而当init进程退出时，系统也将退出。

> 容器文件系统仅提供用户空间隔离，容器文件系统与主机文件系统共享相同的内核。

由于PID为1的进程的特殊性，所以每个PID namespace的第一个进程的PID都是1。当这个进程运行停止后，内核将会给这个namespace里的所有其他进程发送SIGKILL信号，致使其他所有进程都停止，于是namespace被销毁掉。

## PID Namespace

进入第2.2节提及的`/root/ubuntu/fs-go`联合挂载点，切换根文件系统

```shell
unshare --mount --fork /bin/bash # 新建一个命名空间
mkdir -p /root/ubuntu/fs-go/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/fs-go/ /root/ubuntu/fs-go/put_old # 切换根文件系统
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
top # 输出新旧文件系统下所运行的进程信息
```

创建命名空间命令添加`--pid`参数，创建一个`mount + pid`新命名空间，完成根文件系统和进程空间的隔离。

```shell
unshare --mount --pid --fork /bin/bash # 新建一个命名空间
mkdir -p /root/ubuntu/fs-go/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/fs-go/ /root/ubuntu/fs-go/put_old # 切换根文件系统
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
mount -t proc proc /proc # PID namespace变更后重新把内存中的系统状态、进程信息挂载到/proc
top # 只输出新文件系统下所运行的进程信息
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    1 root      20   0  121448   6160   1756 S   0.0   0.1   0:00.11 bash
   43 root      20   0    6088   1820   1320 R   0.0   0.0   0:00.00 top
```

>/proc 文件系统是一种内核和内核模块用来向进程发送信息的机制(所以叫做/proc)。这个伪文件系统让你可以和内核内部数据结构进行交互，获取有关进程的有用信息，在运行中改变设置(通过改变内核参数)。 与其他文件系统不同，/proc 存在于内存之中而不是硬盘上。

