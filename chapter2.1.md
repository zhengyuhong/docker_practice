# 独立文件系统

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
                "MergedDir": "/var/lib/docker/overlay2/2accf1290c8354faf205d4d0f7e1fb3e109d3c9d58c2fa2e470e06698aea9152/merged"
```

````shell
cd /var/lib/docker/overlay2/2accf1290c8354faf205d4d0f7e1fb3e109d3c9d58c2fa2e470e06698aea9152/merged
ls
bin  dev  etc  FLAG  home  lib  lib64  proc  root  sys  tmp  usr  var
````

## chroot

每一个容器都有一套独立文件系统，我们还是以`busybox`为例子，这里不打算重新编译生成`busybox`文件系统，我们直接通过`docker save`命令导出镜像`busybox:glibc`内部的文件系统

```shell
docker save busybox:glibc -o busybox.tar
tar xf busybox.tar
tree -L 2
.
├── a046d8e2afb014ba30903e32d369802841c42345893ea1e4debbb51f201dc9c0
│   ├── json
│   ├── layer.tar
│   └── VERSION
├── busybox.tar
├── c1d66042061bc360cc03a574bf71054eab1ea4577be8340dca85d32e975b3977.json
├── manifest.json
└── repositories

1 directory, 7 files
mkdir fs
cd fs
tar xf ../*/layer.tar
ls
bin  dev  etc  home  lib  lib64  root  tmp  usr  var
```

现在我们已经得到busybox的文件系统，存放在fs目录，通过`chroot`可以实现`docker container run busybox:glibc sh`的功能。

>chroot 命令用来在指定的根目录下运行指令。chroot，即 change root directory （更改 root 目录）。在 linux 系统中，系统默认的目录结构都是以/，即是以根 (root) 开始的。而在使用 chroot 之后，系统的目录结构将以指定的位置作为/位置。
>
>在经过 chroot 命令之后，系统读取到的目录和文件将不在是旧系统根下的而是新根下（即被指定的新的位置）的目录结构和文件

```
chroot --help
Usage: chroot [OPTION] NEWROOT [COMMAND [ARG]...]
```

进入上述fs目录后执行，`chroot . sh`即可切换根目录，实现容器文件系统隔离，每一个容器拥有独立的文件系统环境。