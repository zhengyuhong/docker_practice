# 3.6 User Namespace

## root

在使用Docker时，可以在非root的普通用户下启动容器，并且在容器内容允许应用、程序以root帐号形式进行运行。

```
whoami # 输出当前用户名
zhengyuhong
docker container run -it ubuntu:20.04 bash
top
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    1 root      20   0    4232   2184   1616 S   0.0   0.0   0:00.05 bash
   27 root      20   0    6088   1808   1308 R   0.0   0.0   0:00.00 top
```

实际上Docker通过 user namespace 技术，把宿主机中的一个非root的普通用户映射到容器中的 root 用户，让容器内部的root用户对容器内部资源具有root权限，但是对于容器外部非隔离的资源仅仅拥有普通用户权限。

## namespace

**User Namespace** 是 Linux 3.8 新增的一种 namespace，用于隔离安全相关的资源，包括 user IDs and group IDs**，**keys**, 和 **capabilities。同样一个用户的 user ID 和 group ID 在不同的 user namespace 中可以不一样。简单来说，一个用户可以在一个 user namespace 中是普通用户，但在另一个 user namespace 中是超级用户。**User Namespace**是唯一一种不要求root权限就可以创建的namespace。

```
whoami
zhengyuhong
unshare --user -r /bin/bash # 新建user namespace 
whoami # 已成功将普通用户zhengyuhong映射为超级用户root
root 
```

映射为超级用户root，就按照3.1~3.4隔离根文件系统、挂载目录、网络设备、进程空间、主机域名空间。

```
unshare --mount --fork /bin/bash # 新建一个命名空间
mkdir -p /root/ubuntu/fs-go/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/fs-go/ /root/ubuntu/fs-go/put_old # 切换根文件系统
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
mount # 打印输出只与当前根文件系统相关的挂载目录
```

