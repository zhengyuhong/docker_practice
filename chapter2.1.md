# 2.1 根文件系统

## BusyBox

使用过Docker容器的朋友应该都知道，容器内部有一套独立文件系统，以````busybox````为例，进入容器执行````ls````可见容器内部的文件系统

> BusyBox是一个集成了一百多个最常用 Linux 命令和工具（如 cat、echo、grep、mount、telnet 等）的精简工具箱，它只需要几 MB 的大小，很方便进行各种快速验证，被誉为"Linux 系统的瑞士军刀"。
>
> busybox:glibc 支持glibc的极简Linux镜像

```shell
docker pull busybox:glibc
docker container run -it --name busybox busybox:glibc sh
/ # ls
bin    dev    etc    home   lib    lib64  proc   root   sys    tmp    usr    var
/ # touch FLAG
```

在容器内部执行````touch FLAG````，并如下获取容器内部的文件系统路径，在宿主机进入容器内部的`busybox`文件系统，````ls````即可看到刚刚创建的````FLAG````文件，由此可见容器的文件系统是挂载在宿主机文件系统中。

```shell
docker inspect busybox|grep MergedDir
                "MergedDir": "/var/lib/docker/overlay2/85c7690f3c0d3a0d0db77322591ee4d675267d12873861298f644131d5cb40f0/merged",
```

````shell
cd /var/lib/docker/overlay2/85c7690f3c0d3a0d0db77322591ee4d675267d12873861298f644131d5cb40f0/merged
ls
bin  dev  etc  FLAG  home  lib  lib64  proc  root  sys  tmp  usr  var
````

## chroot

每一个容器都有一套独立文件系统，我们还是以`busybox`为例子，这里不打算重新编译生成`busybox`文件系统，我们直接通过`docker save`命令导出镜像`busybox:glibc`内部的文件系统

```shell
mkdir -p ~/busybox
cd ~/busybox
docker save busybox:glibc -o busybox.tar
tar xf busybox.tar
tree -L 2
.
├── 72dcce2ea38424ed6fb5c879983c911ae661becde8bcd6589ef4ee6dce466132
│   ├── json
│   ├── layer.tar
│   └── VERSION
├── bd6b8f9ad6c6b5dbc05f73ece701390c3958d9815eebbd8ac466de5d3ec81dcb.json
├── busybox.tar
├── manifest.json
└── repositories

1 directory, 7 files

mkdir rootfs
cd rootfs
tar xf ../*/layer.tar
ls
bin  dev  etc  home  lib  lib64  root  tmp  usr  var
```

现在我们已经得到busybox的文件系统，存放在rootfs目录，通过`chroot . sh`可以实现`docker container run busybox:glibc sh`的功能。

>chroot 命令用来在指定的根目录下运行指令。chroot，即 change root directory （更改 root 目录）。在 linux 系统中，系统默认的目录结构都是以/，即是以根 (root) 开始的。而在使用 chroot 之后，系统的目录结构将以指定的位置作为/位置。
>
>在经过 chroot 命令之后，系统读取到的目录和文件将不在是旧系统根下的而是新根下（即被指定的新的位置）的目录结构和文件

```
chroot --help
Usage: chroot [OPTION] NEWROOT [COMMAND [ARG]...]
```

进入上述fs目录后执行，`chroot . sh`即可切换根目录，实现容器文件系统隔离，每一个容器拥有独立的文件系统环境。

## pivot_root

除了 `chroot`，Linux 还提供了 `pivot_root` 系统调用能够将把整个根文件系统（**rootfs**）切换到一个新的根目录。

- **chroot**，把当前进程切换到新的根目录，类似创建一个沙盒，让当前进程运行在沙盒之内，其他进程运行在旧根文件系统下。
- **pivot_root**，把整个根文件系统（**rootfs**）切换到一个新的根目录，移除对旧根文件系统的依赖，以便于对旧根文件系统进行`umount`操作。

相比 `chroot`，`pivot_root` 系统调用配合`Mount Namspace` 更加安全，Docker容器运行时会优先使用这种方式，但 `chroot` 也是一种可选的方式。在 `runC` 的实现中有以下代码片段：

```
if config.NoPivotRoot {
		err = msMoveRoot(config.Rootfs)
	} else if config.Namespaces.Contains(configs.NEWNS) {
		err = pivotRoot(config.Rootfs)
	} else {
		err = chroot()
	}
```

第一行是由用户指定的「不切换根目录」的选择分支，其余分支代表两种切换根目录的方式：

- 如果创建了新的 `Mount Namespace` ，将使用 `pivot_root` 系统调用。
- 如果没有创建新的 `Mount Namespace`，直接使用 `chroot` 。

关于使用`pivot_root` 切换根文件系统，会在下一章`Namespace`引入`Mount Namespace`后展开叙述，本章节就以`chroot`作为文件系统隔离实现方案。







