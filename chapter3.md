# 第3章 Linux Namespace

**Namespace** 是 Linux 内核的一个特性，该特性可以实现在同一主机系统中，对进程 ID、主机名、用户 ID、文件名、网络和进程间通信等资源的隔离。**Docker** 利用 Linux 内核的 Namespace 特性，实现了每个容器的资源相互隔离，从而保证容器内部只能访问到自己 Namespace 的资源。

最新的 Linux 5.6 内核中提供了 8 种类型的 Namespace：

| Namespace名称                    | 作用                                      | 内核版本 |
| :------------------------------- | :---------------------------------------- | :------- |
| mount（mnt）                     | 隔离挂载点                                | 2.4.19   |
| Process ID （pid）               | 隔离进程 ID                               | 2.6.24   |
| Network (net)                    | 隔离网络设备、端口号等                    | 2.6.29   |
| Interprocess Communication (ipc) | 隔离 System V IPC 和 POSIX message queues | 2.6.19   |
| UTS Namespace(uts)               | 隔离主机名和域名                          | 2.6.19   |
| User Namespace (user)            | 隔离用户和用户组                          | 3.8      |
| Control group (cgroup) Namespace | 隔离 Cgroups 根目录                       | 4.6      |
| Time Namespace                   | 隔离系统时间                              | 5.6      |

Linux 内核提供了8种 Namespace，目前 Docker 只使用了其中的前6 种，分别为**Mount Namespace**、**PID Namespace**、**Net Namespace**、**IPC Namespace**、**UTS Namespace**、**User Namespace**。

接下来通过一个命令行工具 unshare来演示操作Docker 使用的 6 种 Namespace 。unshare 是 util-linux 工具包中的一个工具，CentOS 7 系统默认已经集成了该工具，使用 unshare 命令可以实现创建并访问不同类型的 Namespace。

- 第3.1节 [Mount Namespace](chapter3.1.md)
- 第3.2节 [PID Namespace](chapter3.2.md)
- 第3.3节 [UTS Namespace](chapter3.3.md)
- 第3.4节 [Net Namespace](chapter3.4.md)
- 第3.5节 [User Namespace](chapter3.5.md)
- 第3.6节 [IPC Namespace](chapter3.6.md)

