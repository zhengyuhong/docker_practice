# 3.6 IPC Namespace

IPC Namespace提供进程间通信(信号量、消息队列、共享内存)的隔离功能，在一个IPC Namespace里面创建的IPC object对该Namespace内的所有进程可见，但是对其他Namespace不可见，这样就使得不同Namespace之间的进程不能通过IPC通信，就像是在不同的系统里一样。

在日常工作实践中，进程间通信基本上是基于数据库、消息队列（kafak、rabbitmq）、套接字，没有用过IPC进行进程间通信，以下通过PID Namespace 和 IPC Namespace 一起使用，实现同一 IPC Namespace 内的进程彼此可以通信，不同 IPC Namespace 的进程却不能通信。

通过一个实例来验证下IPC Namespace的作用，首先我们使用 unshare 命令来创建一个 IPC Namespace：

```
unshare --ipc --pid --fork /bin/bash
```

需要借助两个命令来实现对 IPC Namespace 的验证

- ipcs -q 命令：用来查看系统间通信队列列表
- ipcmk -Q 命令：用来创建系统间通信队列

首先使用 ipcs -q 命令查看一下当前 IPC Namespace 下的系统通信队列列表：

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

可以看到当前无任何系统通信队列，使用 ipcmk -Q 命令创建一个系统通信队列：

```shell
ipcmk -Q
Message queue id: 0
```

再次使用 ipcs -q 命令查看当前 IPC Namespace 下的系统通信队列列表：

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0xdd3ced96 0          root       644        0
```

可以看到已经成功创建了一个系统通信队列。然后我们新打开一个命令行窗口，使用ipcs -q 命令查看一下主机的系统通信队列：

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

通过以上可以发现，在单独的 IPC Namespace 内创建的系统通信队列在主机上无法看到。即 IPC Namespace 实现了系统通信队列的隔离。

## 引用

[Linux Namespace : IPC](https://www.cnblogs.com/sparkdev/p/9400673.html)

[System V IPC 之共享内存](https://www.cnblogs.com/sparkdev/p/8656898.html)

[System V IPC 之信号量 ](https://www.cnblogs.com/sparkdev/p/8692567.html)

[System V IPC 之消息队列 ](https://www.cnblogs.com/sparkdev/p/8716710.html)

