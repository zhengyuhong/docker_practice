# 3.4 Net Namespace

## echo

 宿主机启动如下`go run echo.go`服务

```
cd /root
cat <<EOF > echo.go
package main

import (
    "net/http"
    "bytes"
    "fmt"
)

func EchoHandler(writer http.ResponseWriter, request *http.Request) {
    buf := bytes.Buffer{}
    buf.ReadFrom(request.Body)
    fmt.Println(buf.String())
    fmt.Fprintf(writer, buf.String())
}

func main() {
    http.HandleFunc("/echo", EchoHandler)
    addr := fmt.Sprintf("0.0.0.0:8888")
    fmt.Println("listen on 0.0.0.0:8888")
    http.ListenAndServe(addr, nil)
}
EOF
go run echo.go &
listen on 0.0.0.0:8888
```

启动一个ubuntu容器，安装`go1.18.4`，已安装可忽略此步骤

```
docker container run -it ubuntu:20.04 bash
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

容器启动如下`go run echo.go`服务

```
cd /root
cat <<EOF > echo.go
package main

import (
    "net/http"
    "bytes"
    "fmt"
)

func EchoHandler(writer http.ResponseWriter, request *http.Request) {
    buf := bytes.Buffer{}
    buf.ReadFrom(request.Body)
    fmt.Println(buf.String())
    fmt.Fprintf(writer, buf.String())
}

func main() {
    http.HandleFunc("/echo", EchoHandler)
    addr := fmt.Sprintf("0.0.0.0:8888")
    fmt.Println("listen on 0.0.0.0:8888")
    http.ListenAndServe(addr, nil)
}
EOF
go run echo.go &
listen on 0.0.0.0:8888
```

宿主机、容器的echo服务均监听在`8888`端口上，可见容器与宿主机的端口是相互隔离的。

## ip addr

Net Namespace 是用来隔离网络设备、IP 地址和端口等信息的，Net Namespace 可以让每个进程拥有自己独立的 IP 地址，端口和网卡信息。例如主机 IP 地址为 10.12.186.188 ，容器内可以设置独立的 IP 地址为 192.168.1.1。

使用`ip addr`查询主机网络设备

```
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether fa:27:00:03:12:15 brd ff:ff:ff:ff:ff:ff
    inet 10.12.186.147/22 brd 10.12.187.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f827:ff:fe03:1215/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:60:62:f5:8b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:60ff:fe62:f58b/64 scope link
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether ce:e4:28:f8:aa:0c brd ff:ff:ff:ff:ff:ff
    inet 10.244.0.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::cce4:28ff:fef8:aa0c/64 scope link
       valid_lft forever preferred_lft forever
30: vethdc59069@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 76:69:bb:35:82:93 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::7469:bbff:fe35:8293/64 scope link
       valid_lft forever preferred_lft forever
34: vethc329122@if33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether ae:c8:cd:89:b9:e4 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::acc8:cdff:fe89:b9e4/64 scope link
       valid_lft forever preferred_lft forever
```

进入ubuntu容器，安装`apt-get install iproute2`

```
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
apt-get update
apt-get install net-tools iproute2
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
40: eth0@if41: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

可以看到，除了环回地址127.0.0.1，宿主机网络与容器网络设备并无相同部分。

## Net Namespace

按照第3.1节进入fs-go联合文件系统，创建Net Namespace，使用`ifconfig`查看新namespace下发现没有网络设备，可理解为是一个没有网络的文件系统，等后续章节再补充如何让当前namespace和外部环境联网互通。

```
unshare --mount --fork /bin/bash # 新建一个命名空间
mkdir -p /root/ubuntu/fs-go/put_old # 用于挂载旧根文件系统
pivot_root /root/ubuntu/fs-go/ /root/ubuntu/fs-go/put_old # 切换根文件系统
umount -l /put_old # 隐藏旧根文件系统的挂载，/put_old变成空目录
rmdir /put_old # 删除空目录
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apt-get update 
apt-get install net-tools iproute2
cd /
unshare --net --fork /bin/bash # 新建一个命名空间
ip addr # 只有本地环回地址，并且这个接口是处于关闭状态的。
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

启动如下`go run echo.go`服务

```
cd /root
cat <<EOF > echo.go
package main

import (
    "net/http"
    "bytes"
    "fmt"
)

func EchoHandler(writer http.ResponseWriter, request *http.Request) {
    buf := bytes.Buffer{}
    buf.ReadFrom(request.Body)
    fmt.Println(buf.String())
    fmt.Fprintf(writer, buf.String())
}

func main() {
    http.HandleFunc("/echo", EchoHandler)
    addr := fmt.Sprintf("0.0.0.0:8888")
    fmt.Println("listen on 0.0.0.0:8888")
    http.ListenAndServe(addr, nil)
}
EOF
go run echo.go &
listen on 0.0.0.0:8888
```

宿主机、新net namespace的echo服务均监听在`8888`端口上，可见两者网络环境是相互隔离的，在搭建容器网络之前两者甚至是不连通的。

