# 第5章 Cgroup

## 为什么需要Cgroup

第2章讲述容器的文件系统（rootfs）的隔离、共享方案。

第3章讲述容器如何使用Linux Namespace，实现每个容器的资源相互隔离，保证容器内部只能访问到自己 Namespace 的资源。

第4章补充讲述如何给相互隔离的net namespace添加虚拟网络实现相互通信。

容器运行所需的rootfs、mount namespace、pid namespace、uts namespace、net namespace、user namespace、ipc namespace已有前几个技术进行资源隔离，但是容器运行所需的cpu、内存、磁盘IO、网络IO等并没有进行资源限制，一旦某一个容器占用过多资源，必然会影响其他容器的运行状态，于是就有对进程进行分组、约束的需求。

在 Linux 里，一直以来就有对进程进行分组的概念和需求，比如session group， progress group等，后来随着人们对这方面的需求越来越多，比如需要追踪一组进程的内存和 IO 使用情况等，于是出现了 Cgroup，用来统一将进程进行分组，并在分组的基础上对进程进行监控和资源控制管理等。

实现资源限制的技术有多种，其中Cgroup是内核实现的，它更轻量、更高效、对内核的热点路径影响最小。

## 什么是Cgroup

Cgroup 是 Control Group 的缩写，提供对一组进程，及未来子进程的资源**限制、控制、统计**能力，包括CPU、内存、磁盘、网络。

- 限制：限制的资源最大使用量阈值。比如不能超过128MB内存，CPU使用率不得超过50%。
- 控制：超过资源使用最大阈值时，进程会被控制，不任由它发展。比如cgroup内所有tasks的内存使用量超过阈值的结果就是被KILL，CPU使用率不得超过设定值。
- 统计：统计资源的使用情况等指标。比如cgroup内tasks的内存使用量，占用CPU的时间。

Cgroup 包含3个组件：

- cgroup ：一组进程，可以加上subsystem
- subsystem ：一组资源控制模块，CPU、内存、磁盘、网络
- hierarchy ： 把一组cgroup串成树状结构，这样就能实现cgroup的继承，站在前人的基础之上，免去重复的配置。

