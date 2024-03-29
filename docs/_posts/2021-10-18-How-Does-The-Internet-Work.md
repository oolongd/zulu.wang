---
category: network
tags:
  - TCP/IP
  - network
date: 2021-10-18
title: 互联网是如何运作的？
vssue-title: How does the Internet work?
---

![202107141440501ZZN5K](https://cdn.jsdelivr.net/gh/oolongd/assets@main/img/202107141440501ZZN5K.jpg)

## 什么是互联网

在探讨互联网是什么之前，我们必须清楚什么是“网络”。网络是一组彼此连接并能够相互发送数据的计算机。计算机网络很像一个社交圈，即一群互相认识、定期交换信息并共同协调活动的人。

由于网络内的计算机相互连接，并且这些网络也都相互连接，因此通过互联网，一台计算机可以与远方网络中的另一台计算机通信。这样，使得世界各地的计算机之间快速交换信息成为可能。

计算机通过线缆、无线电波和其他类型的网络基础设施相互连接并与互联网连接。通过互联网发送的所有数据都被转换为光或电脉冲（也称为“位”），然后由接收计算机进行解释。线缆和无线电波以光速传导这些位。一次可以越过这些线缆的位数越多，互联网的运行速度就越快。



## 什么是分布式网络？为什么这个概念对互联网很重要？

互联网没有控制中心。相反，它是一种分布式网络系统；也就是说，它不依赖于任何单独的计算机。任何计算机或硬件只要能以正确方式（例如，使用正确的网络协议）发送和接收数据，都可成为互联网的一部分。

互联网的分布式特性使其具有弹性。计算机、服务器和其他网络硬件可以随时连入互联网或断开连接，不会影响互联网的功能；计算机则不一样，缺少某一组件时可能就完全无法运行。大规模情况下也是如此：如果服务器、整个数据中心或整个地区的数据中心发生故障，互联网的其余部分仍然可以正常运行（速度可能会变慢）。



## 当今互联网是如何运作的？

互联网运作的基本原理包含两个主要概念：*数据包*和*协议*。

### 数据包

当数据通过互联网发送时，它首先分解为较小的数据包，接着转换为位。数据包通过各种网络设备（如路由器和交换机）路由到其目的地。当数据包到达目的地时，接收设备会按顺序重组数据包，然后可以使用或显示数据。

### 协议

连接两台可能使用不同硬件并运行不同软件的计算机，是互联网缔造者必须解决的主要挑战之一。这需要使用所有连接的计算机都可理解的通信技术，就像在世界上不同地区长大的两个人可能需要说一种通用语言才能互相理解一样。

这个问题可标准化协议解决。在网络中，协议是执行某些操作和格式化数据的标准化方法，以便两个或更多设备能够相互通信并相互理解。

所有连入互联网的计算机和其他设备都可以解释和理解这些协议，所以无论连接的是什么，互联网都可以正常工作。



## 促成互联网运作的物理基础设施有哪些？

### 路由器

路由器是一种网络硬件，负责将数据包转发到目的地。路由器连接到两个或多个 IP 网络或子网，并根据需要在它们之间传递数据包。路由器在家庭和办公室中用于建立本地网络连接并接入互联网。功能更强大的路由器在整个 Internet 上运行，从而帮助数据包到达目的地。

#### 什么是路由

请参考下图。对于要从计算机 A 到达计算机 B 的数据包，它应该通过网络 1、3 和 5 还是通过网络 2 和 4？数据包通过网络 2 和 4 的路径会更短，但是网络 1、3 和 5 在转发数据包时可能比 2 和 4 更快。这些都是网络路由器不断做出的选择类型。

#### 路由器如何工作

路由器通过参考内部路由表来决定如何沿网络路径路由数据包。路由表记录了数据包应到达路由器负责的每个目的地的路径。类似于列车时刻表，乘客会查阅时刻表以决定搭乘哪趟列车。路由表也是如此，但是用于网络路径而不是列车。

路由器以下列方式工作：路由器接收到数据包时，会读取数据包的标头以查看其预期的目的地。然后，根据路由表中的信息确定将数据包路由到何处。这种方式类似于快递系统。在每个快递中转站，中转站的工作人员或机器都要判断我的包裹应该用什么方式，走哪条路线才能高效、准确地到达目的地点，然后将我的包裹发送出去。

路由表可以是静态的，也可以是动态的。静态路由表不变化。网络管理员手动设置静态路由表。除非管理员手动更新这些表，否则，路由表完全可以确定数据包在网络上的路由。

动态路由表会自动更新。动态路由器使用各种路由协议来确定最短和最快的路径。它们还根据数据包到达目的地所需的时间做出决定 。因为最短的路径并不见得是最优的传输路径。

### 交换机

交换机主要用来连接网络内的设备，并将数据包转发给这些设备或者为这些设备转发数据。与路由器不同的是，交换机只将数据发送给单个设备，而不是拥有多个设备的网络。

#### 交换机如何工作

交换机工作于 OSI  中的数据链路层。交换机内部的 CPU 会在每个端口成功连接时，通过将 MAC 地址和端口对应，形成一张 MAC 表。在今后的通讯中，发往该MAC地址的数据包将仅送往其对应的端口，而不是所有的端口。

#### 什么是二层交换机，什么是三层交换机？

交换机可以在 OSI 第二层（数据链路层）或第三层（ 网络层 ）上运行。 二层交换机根据目标 MAC 地址转发数据，而三层交换机则根据目标 IP 地址转发数据。

#### 交换机和路由器有什么区别？

* 工作层次
  - 交换机主要工作在数据链路层
  - 路由器工作在网络层
* 转发依据
  - 交换机转发所依据的对象时：MAC 地址
  - 路由转发所依据的对象是：IP 地址
* 主要功能
  - 交换机主要用于组建局域网
  - 路由主要功能是将由交换机组好的局域网相互连接起来，或者接入Internet。
  - 路由器的功能比交换机的功能更加强大，交换机能做的，路由器都能做。

打一个通俗的比喻，路由器就像是出租车，上车后司机司机需要知道你的目的地（IP 地址），然后司机来负责选择线路到达目的地。到达目的地的线路可能根据实际的交通情况有所变化。交换机就像地铁，从 A 到 B 的线路是固定的，只要你选择了到达地址（MAC），你就只能乘坐某条固定的线路，地铁司机也不会问你去哪里。

### 服务器

服务器是一种特殊的高性能计算机，除了为应用程序和数据库提供计算能力外，还可以存储内容（网页、图像和视频等）并为用户提供相应的服务。



