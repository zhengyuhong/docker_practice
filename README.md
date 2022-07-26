# Docker 从应用到实现

## 前言

我在2014年研究生阶段就开始接触Docker，但是仅仅知道这个名词而已，并没有真正使用起来，更不用说了解其中实现原理。这两年，由于工作上的业务落地部署需求，我也开始使用Docker容器引擎、K8S容器编排工具，也逐渐了解到Docker底层相关技术Cgroup、Namespace、OverlayFS，但是再深入一些，Docker是如何组合这些技术、是怎么从0到1创建容器的？这些我也是一头雾水，为了理解和掌握更多容器底层技术，我从使用Docker的角度出发，尝试自己动手去开发实现自己看到的Docker容器隔离、资源限制等功能。

# 目录

- 第1章 [什么是 Docker](chapter1.md)
  - 第1.1节 [环境准备](chapter1.1.md)

- 第2章 [文件系统](chapter2.md)
  - 第2.1节 [根文件系统](chapter2.1.md)
  - 第2.2节 [联合文件系统](chapter2.2.md)
- 第3章 [Linux Namespace](chapter3.md)
  - 第3.1节 [Mount Namespace](chapter3.1.md)
  - 第3.2节 [PID Namespace](chapter3.2.md)
  - 第3.3节 [UTS Namespace](chapter3.3.md)
  - 第3.4节 [Net Namespace](chapter3.4.md)
  - 第3.5节 [User Namespace](chapter3.5.md)
  - 第3.6节 [IPC Namespace](chapter3.6.md)
- 第4章 [虚拟网络](chapter4.md)
  - 第4.1节 [桥接网络模型](chapter4.1.md)
- 第5章

