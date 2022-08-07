# 2.2 联合文件系统

## UnionFS

在企业软件生产环境中，一台宿主机上运行了几十上百个包含文件系统的容器，每个容器里都有一份独立文件系统。无论 ubuntu 是 72MB 还是 centos 231MB 的大小都非常庞大，每次我们想从这些镜像创建一个容器时，分配这么多空间是非常昂贵的。Docker实现中借助Linux的`UnionFS`，只需要在文件系统镜像之上创建一个轻量可写文件层，底层的只读文件系统层可以在所有容器之间共享。

>**UnionFS**联合文件系统是一种**分层、轻量级**并且**高性能**的文件系统，具备如下特性：
>
>- 联合挂载：将多个目录按层次组合，一并挂载到一个联合挂载点。
>
>- 写时复制：对联合挂载点的修改不会影响到底层的多个目录，而是使用其他目录记录修改的操作。

## OverlayFS

`OverlayFS`是其中一种联合文件系统，`OverlayFS`构建于其他文件系统之上，`OverlayFS`其实更像是个挂载系统，功能是把不同的文件系统挂载到统一的路径。接下来以`ubuntu`为基础文件系统，构建分别包含`go`,`python`联合文件系统。

### ubuntu:20.04 基础文件系统

类似上一节通过`docker save`命令导出`ubuntu`文件系统

```shell
mkdir /root/ubuntu
cd /root/ubuntu
docker pull ubuntu:20.04
docker save ubuntu:20.04 -o ubuntu.tar
tar xf ubuntu.tar
tree -L 2
.
├── 3bc6e9f30f51d2bbf9307fc9d0bdfc30caa38cf4e3b05a714230f9a9b3381d84.json
├── ce0735ccbbc5039603124748c7cb624862ef80812b9787522c0f4aea9406061b
│   ├── json
│   ├── layer.tar
│   └── VERSION
├── manifest.json
├── repositories
└── ubuntu.tar
1 directory, 7 files

mkdir rootfs
cd rootfs
tar xf ../*/layer.tar
ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### ubuntu:python 文件系统

接下来以`/root/ubuntu/rootfs/`为基础文件系统，在此之上添加`python`环境

```shell
mkdir -p /root/ubuntu/rootfs-python
mkdir -p /root/ubuntu/rootfs-python/diff # 存放增量的python文件系统
mkdir -p /root/ubuntu/rootfs-python/merged # 挂载基础文件系统rootfs + 增量python文件系统的联合文件系统
mkdir -p /root/ubuntu/rootfs-python/work # 用于存放挂载后的临时文件和间接文件
```

>mount -t overlay overlay -o lowerdir=lower1:lower2:...,upperdir=upper,workdir=work merged
>
>- lowerdir：表示较为底层的目录（支持多个，越底层越靠右侧），修改联合挂载点不会影响到lowerdir。
>- upperdir：表示较为上层的目录，修改联合挂载点会在upperdir同步修改。
>- merged：是lowerdir和upperdir合并后的联合挂载点。
>- workdir：用来存放挂载后的临时文件与间接文件。

```shell
mount -t overlay overlay -o lowerdir=/root/ubuntu/rootfs/,upperdir=/root/ubuntu/rootfs-python/diff,workdir=/root/ubuntu/rootfs-python/work /root/ubuntu/rootfs-python/merged
mount|grep rootfs-python # 查看挂载情况
overlay on /root/ubuntu/rootfs-python/merged type overlay (rw,relatime,lowerdir=/root/ubuntu/rootfs/,upperdir=/root/ubuntu/rootfs-python/diff,workdir=/root/ubuntu/rootfs-python/work)
```

进入`/root/ubuntu/rootfs-python/merged`联合挂载点，切换根目录，更新apt源，安装python

```shell
cd /root/ubuntu/rootfs-python/merged
chroot . sh
mount -t proc proc /proc
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat <<EOF > /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt-get update
apt-get install python
```

安装完成后，执行`exit`，退出`chroot`，查看`du -sh *`可见`/root/ubuntu/rootfs-python/merged`联合挂载系统新增的python软件包已添加到`root/ubuntu/rootfs-python/diff`，而`/root/ubuntu/rootfs`是基础文件系统，只读不写，保持不变。

```
cd /root/ubuntu/
du -sh rootfs rootfs-python/diff rootfs-python/work rootfs-python/merged
78M	rootfs
103M	rootfs-python/diff
8.0K	rootfs-python/work
180M	rootfs-python/merged
```

### ubuntu:go 文件系统

继续以`/root/ubuntu/rootfs/`为基础文件系统，在此之上添加`go`环境

```
mkdir -p /root/ubuntu/rootfs-go/diff # 存放增量的go文件系统，初始化为空目录
mkdir -p /root/ubuntu/rootfs-go/merged # 挂载基础文件系统rootfs + 增量go文件系统的联合文件系统
mkdir -p /root/ubuntu/rootfs-go/work # 用于存放挂载后的临时文件和间接文件
```

```
mount -t overlay overlay -o lowerdir=/root/ubuntu/rootfs/,upperdir=/root/ubuntu/rootfs-go/diff,workdir=/root/ubuntu/rootfs-go/work /root/ubuntu/rootfs-go/merged
mount|grep rootfs-go # 查看挂载情况
```

进入`/root/ubuntu/rootfs-go/merged`联合挂载点，切换根目录，更新apt源，安装wget，再下载go二进制包

```
cd /root/ubuntu/rootfs-go/merged
chroot . sh
mount -t proc proc /proc
cp /etc/apt/sources.list /etc/apt/sources.list.bak
cat <<EOF > /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt-get update
apt-get install wget
cd /tmp
wget --no-check-certificate https://go.dev/dl/go1.18.4.linux-amd64.tar.gz
tar xfz go1.18.4.linux-amd64.tar.gz && rm go1.18.4.linux-amd64.tar.gz
mkdir -p /usr/local/bin
mv /tmp/go /usr/local/bin/go1.18.4
cd /usr/local/bin/
ln -s go1.18.4/bin/go .
echo "export GOROOT=/usr/local/bin/go1.18.4" >> /etc/bash.bashrc
```

安装完成后，执行`exit`，退出`chroot`，查看`du -sh *`可见`/root/ubuntu/rootfs-go/merged`联合挂载系统新增的python软件包已添加到`root/ubuntu/rootfs-go/diff`，而`/root/ubuntu/rootfs`是基础文件系统，只读不写，保持不变。

```
cd /root/ubuntu
du -sh rootfs rootfs-go/diff rootfs-go/work rootfs-go/merged
78M	rootfs
522M	rootfs-go/diff
8.0K	rootfs-go/work
599M	rootfs-go/merged
```

### ubuntu:go_python 文件系统

假设现在需要一个`go+python`的ubuntu文件系统，那可以基于上述`ubuntu:go`添加增量的`python`文件系统（当然也可以基于`ubuntu:python`添加增量的`go`文件系统）。

```
mkdir -p /root/ubuntu/rootfs-go_python/diff # 存放增量的python文件系统，初始化为空目录
mkdir -p /root/ubuntu/rootfs-go_python/merged # 挂载基础文件系统rootfs + 增量go文件系统的联合文件系统 + 增量python文件系统的联合文件系统
mkdir -p /root/ubuntu/rootfs-go_python/work # 用于存放挂载后的临时文件和间接文件
```

```
mount -t overlay overlay -o lowerdir=/root/ubuntu/rootfs-go/diff:/root/ubuntu/rootfs/,upperdir=/root/ubuntu/rootfs-go_python/diff,workdir=/root/ubuntu/rootfs-go_python/work /root/ubuntu/rootfs-go_python/merged
mount|grep rootfs-go_python # 查看挂载情况
overlay on /root/ubuntu/fs-go_python type overlay (rw,relatime,lowerdir=/root/ubuntu/go/:/root/ubuntu/fs,upperdir=/root/ubuntu/go_python,workdir=/root/ubuntu/fs-go_python-work)
```

仔细观察上述`mount`命令，`lowerdir`包含两个路径`/root/ubuntu/rootfs-go/diff:/root/ubuntu/rootfs/`，越底层越靠右侧。

进入`/root/ubuntu/rootfs-go_python/merged`联合挂载点，切换根目录，安装`python`

```
cd /root/ubuntu/rootfs-go_python/merged
chroot . sh
mount -t proc proc /proc
apt-get install python
```

安装完成后，执行`exit`，退出`chroot`，查看`du -sh *`可见`/root/ubuntu/rootfs-go_python/merged`联合挂载系统新增的python软件包已添加到`root/ubuntu/rootfs-go_python/diff`，而`/root/ubuntu/rootfs-go/diff:/root/ubuntu/rootfs/`是底层基础文件系统，只读不写，保持不变。

```shell
cd /root/ubuntu/
du -sh rootfs rootfs-go/diff rootfs-go_python/diff rootfs-go_python/work rootfs-go_python/merged
78M	rootfs
522M	rootfs-go/diff
34M	rootfs-go_python/diff
8.0K	rootfs-go_python/work
631M	rootfs-go_python/merged
```

## 总结

通过上述几个例子应该明白了**UnionFS**两个特性，通过这两个特性达到`轻量`、`分层`、`复用`的高性能文件系统的效果。

- 联合挂载
  - 通过`mount`把具有依赖关系的目录链(lowerdir=lower1:lower2:...)，逐层挂载合并生成一个联合视图，形成最终的联合文件系统。
  - 联合文件系统的依赖目录链只读不写，不同联合文件系统底层可以依赖相同的一个或者多个底层目录。
- 写时复制
  - 通过`chroot`进入联合文件系统后，写操作都是在`upperdir`层进行的，不会影响`lowerdir。`
  - 在联合文件系统中对`lowerdir`文件进行写操作时，其实是复制`lowerdir`的文件到`upperdir`，在`upperdir`对相应文件进行写操作。
