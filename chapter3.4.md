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

## ifconfig

Net Namespace 是用来隔离网络设备、IP 地址和端口等信息的，Net Namespace 可以让每个进程拥有自己独立的 IP 地址，端口和网卡信息。例如主机 IP 地址为 10.12.186.188 ，容器内可以设置独立的 IP 地址为 192.168.1.1。

使用`ifconfig`查询主机网络设备

```
ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:60ff:fe62:f58b  prefixlen 64  scopeid 0x20<link>
        ether 02:42:60:62:f5:8b  txqueuelen 0  (Ethernet)
        RX packets 19594  bytes 1092565 (1.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25678  bytes 188076306 (179.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.12.186.188  netmask 255.255.252.0  broadcast 10.12.187.255
        inet6 fe80::f827:ff:fe03:1215  prefixlen 64  scopeid 0x20<link>
        ether fa:27:00:03:12:15  txqueuelen 1000  (Ethernet)
        RX packets 8651913  bytes 6487783370 (6.0 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6192601  bytes 2104749962 (1.9 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.244.0.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::cce4:28ff:fef8:aa0c  prefixlen 64  scopeid 0x20<link>
        ether ce:e4:28:f8:aa:0c  txqueuelen 0  (Ethernet)
        RX packets 162290  bytes 79722887 (76.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 270095  bytes 24531255 (23.3 MiB)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 22771362  bytes 4570150757 (4.2 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 22771362  bytes 4570150757 (4.2 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vethc17deb1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::68e8:bfff:fed0:80b7  prefixlen 64  scopeid 0x20<link>
        ether 6a:e8:bf:d0:80:b7  txqueuelen 0  (Ethernet)
        RX packets 19594  bytes 1366881 (1.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 25681  bytes 188076516 (179.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

进入ubuntu容器，安装`apt install net-tools`

```
apt install net-tools
ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 25681  bytes 188076516 (188.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 19594  bytes 1366881 (1.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
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
apt install net-tools
cd /
unshare --net --fork /bin/bash # 新建一个命名空间
ifconfig | wc -l
0
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

