---
title: "从零理解云服务器、计算机网络与 Python 程序部署"
description: "面向 Python 开发者的网络、Linux 运维、Web 服务、API 与自动化运行完整教程"
date: 2026-10-18
lastmod: 2026-10-18
weight: 4
categories:
    - Tutorial
    - Tech Stack
    - Quant
tags:
    - 教程
    - 技术栈
    - 云服务器
    - 网络
    - Linux
    - Python
    - 部署
---

**文档定位：** 系统入门教程 + 原理解释 + 实战手册  
**适用对象：** 会 Python、了解 FastAPI，但对服务器、网络、Linux 运维和服务部署缺乏系统认识的开发者  
**实践项目：** 加密货币行情采集、Parquet 数据存储、DuckDB 查询、FastAPI 因子看板  
**学习目标：** 不仅知道命令怎么执行，还要理解每项技术为什么存在、内部如何工作，以及遇到问题时应该从哪里排查。

---

- 作者：山财小蒋
- 联系方式：2018036661@qq.com
- 创作不易，转载请注明出处，欢迎批评探讨

## 第一章：先建立完整的系统认知

### 1.1 为什么程序部署到云服务器之后会突然变复杂？

在本地开发时，你可能只需要执行：

```python
python main.py
```

或者：

```bash
uvicorn app.main:app --reload
```

然后打开浏览器访问：

```text
http://127.0.0.1:8000
```

程序就可以正常运行。

但当你把程序部署到云服务器之后，会遇到一系列新问题：

- 如何连接远程服务器？
- 为什么服务器有公网 IP，但浏览器仍然访问不了？
- 为什么程序在 SSH 终端关闭后就停止运行？
- 为什么别人无法直接访问 `127.0.0.1:8000`？
- 为什么服务器开放了 8000 端口，仍然可能无法访问？
- 为什么生产环境通常需要 Nginx？
- 为什么有些网站使用域名，而不是直接使用 IP？
- 为什么 HTTPS 需要证书？
- 如何每天凌晨自动更新行情数据？
- 如何保证程序崩溃之后能够自动恢复？
- FastAPI 返回的 JSON 究竟是怎样传到浏览器的？

这些问题看起来分散，实际上可以统一到一个系统模型中理解。

### 1.2 云服务器部署的整体架构

先看一个典型的生产环境架构。

```text
                   用户的电脑 / 浏览器
                           |
                           | ① 输入域名
                           v
                      DNS 域名解析
                           |
                           | ② 获取服务器公网 IP
                           v
                    公网 IP:443 / HTTPS
                           |
                           v
                 云厂商网络安全规则
                    安全组 / 防火墙
                           |
                           v
                    Linux 云服务器
                           |
                           v
                  Nginx 反向代理
                    /           \
                   /             \
        网页与静态资源             API 请求
             |                       |
             |                       v
             |                  FastAPI
             |                       |
             |                 业务处理逻辑
             |                       |
             |             +---------+---------+
             |             |                   |
             |             v                   v
             |         DuckDB              元数据数据库
             |             |
             |             v
             |          Parquet
             |
             v
           浏览器

另一个独立运行链路：

云服务器定时调度器
        |
        v
Python 行情采集任务
        |
        v
交易所 REST API
        |
        v
清洗与标准化
        |
        v
Parquet 分区文件
        |
        v
DuckDB 查询与因子计算
```

这里的每一层都有不同的职责。

| 组件 | 主要职责 | 解决的问题 |
|---|---|---|
| 云服务器 | 提供远程计算资源 | 程序运行在哪里 |
| Linux | 管理计算机硬件资源和进程 | 程序怎样在服务器上运行 |
| 公网 IP | 让外部网络定位服务器 | 请求发往哪台机器 |
| 端口 | 区分同一台机器上的网络服务 | 请求交给哪个服务 |
| DNS | 将域名映射到 IP 地址 | 用户如何通过域名找到服务器 |
| 防火墙、安全组 | 控制哪些网络连接可以进入 | 谁可以访问哪些服务 |
| Nginx | 接收 HTTP/HTTPS 请求并转发 | 如何统一接入、处理 HTTPS 和代理请求 |
| TLS 证书 | 证明服务端身份并建立加密连接 | 如何降低冒充和窃听风险 |
| FastAPI | 实现业务接口 | 如何提供行情和因子数据 |
| 进程管理器 | 监控并管理长期运行的程序 | 程序如何持续运行和自动恢复 |
| 定时调度器 | 在指定时间启动任务 | 如何每天自动采集数据 |
| Parquet | 持久化保存结构化研究数据 | 如何高效保存行情和因子 |
| DuckDB | 查询和分析数据 | 如何读取 Parquet 并执行分析 |
| REST API | 通过 HTTP 交换结构化数据 | 程序如何与交易所等外部系统通信 |

**最重要的认识：云服务器部署不是一个单独的技术，而是多个系统组件共同完成的一条链路。**

例如，浏览器无法访问一个网站，问题可能出在 DNS、云厂商安全组、服务器防火墙、Nginx、FastAPI 或程序自身。只有建立了分层模型，才能准确判断故障位置。

---

## 第二章：什么是云服务器？

### 2.1 服务器到底是什么？

服务器（Server）首先是一种角色，而不一定是一种特殊的硬件。

如果一台计算机运行了一个程序，专门向其他计算机提供服务，那么它就可以称为服务器。

例如：

- 运行网站的计算机，可以叫 Web 服务器。
- 存储和提供数据库查询的计算机，可以叫数据库服务器。
- 接受交易所 API 请求、采集行情的计算机，可以叫数据采集服务器。

你的笔记本电脑同样可以充当服务器。只要其他设备能够通过网络访问它运行的服务，就符合服务器的基本定义。

服务器与客户端（Client）是相对的角色。

例如：

```text
浏览器 = 客户端
FastAPI = 服务端

FastAPI 请求交易所行情：
FastAPI 所在程序 = 客户端
交易所 API = 服务端
```

同一个程序在不同通信关系中，可以同时扮演客户端和服务端。

### 2.2 云服务器是什么？

云服务器是云服务商通过虚拟化技术或其他资源隔离机制提供的可远程使用的计算资源。

你不需要自己购买物理服务器、安装硬盘、接入机房网络，而是向云厂商租用：

- CPU 计算资源
- 内存
- 磁盘空间
- 网络带宽
- 公网或私网网络能力

通常，你可以通过管理控制台创建一台虚拟机，选择操作系统、CPU、内存、磁盘和网络配置。

常见的服务形式包括：

| 类型 | 本质 | 典型用途 |
|---|---|---|
| 云服务器 / 虚拟机 | 一台可管理的远程计算机 | 自定义部署 Python、Nginx、数据库 |
| 容器服务 | 运行相互隔离的容器 | 标准化应用部署 |
| 托管数据库 | 云厂商负责维护数据库基础设施 | 生产数据库 |
| Serverless 函数 | 按事件或请求运行函数 | 事件驱动任务、轻量 API |
| 对象存储 | 提供远程文件存储服务 | 备份、归档、数据集存储 |

对于希望深入理解网络、操作系统和程序部署的开发者，云服务器是很好的起点，因为你可以直接接触系统各层。

### 2.3 云服务器不等于一台随时可以访问的计算机

购买云服务器，只代表你获得了相应的计算资源，并不意味着所有网络服务都已经对外开放。

要让外部用户访问你的程序，至少需要考虑：

1. 服务器是否已经启动。
2. 操作系统是否正常运行。
3. 程序是否启动并监听网络端口。
4. 服务器是否具有可用的公网访问路径。
5. 云厂商安全组是否允许对应流量。
6. 操作系统防火墙是否允许对应流量。
7. DNS 是否指向正确的服务器。
8. 如果使用 HTTPS，TLS 证书和 Web 服务是否配置正确。

这些条件是相互独立的。

例如，服务器拥有公网 IP，但没有放行 TCP 443 端口，外部用户依然无法通过 HTTPS 访问网站。

### 2.4 公有云、私有网络与公网

需要区分三个概念。

**公有云：** 由云厂商提供的计算资源服务。

**公网：** 可以通过互联网进行路由访问的网络。

**私有网络：** 通常指 VPC（Virtual Private Cloud，虚拟私有云）等隔离的云网络环境。

公有云中的服务器可以只有私网 IP，也可以同时拥有私网 IP 和公网 IP。

例如：

```text
公网 IP：203.0.113.10
私网 IP：10.0.1.15
```

上面的 IP 仅为示例。

公网 IP 用于外部网络访问，私网 IP 通常用于同一 VPC 内部的服务器通信。

更合理的生产架构往往是：

```text
互联网
   |
   v
公网入口服务器
   |
   | 私有网络
   v
内部应用服务器
   |
   v
内部数据库
```

不是所有服务器都需要暴露在公网。

对于个人项目，第一阶段可以只使用一台服务器，但仍然应该理解公网入口与内部服务的区别。

### 2.5 选择服务器时要关注什么？

对 Python 数据研究项目而言，重点关注：

- **CPU：** 数据清洗、因子计算和压缩等任务可能消耗大量 CPU。
- **内存：** Pandas、Polars、DuckDB 查询和大型数据集处理可能占用大量内存。
- **磁盘容量：** Parquet 文件、日志、数据库文件和备份都会占用空间。
- **磁盘性能：** 大量文件读取、写入和随机访问可能受 I/O 限制。
- **网络带宽：** 下载行情、访问 API 和向浏览器传输结果都依赖网络。
- **操作系统：** Linux 发行版及其软件包生态会影响部署方式。
- **公网 IP：** 是否需要固定公网入口。
- **地域与网络质量：** 会影响访问交易所、软件源和用户的延迟。
- **费用与备份：** 需要考虑磁盘、快照、流量和额外 IP 等费用。

不要只看 CPU 核数。对于数据研究项目，内存和磁盘往往同样重要。

---

## 第三章：IP 地址、端口与网络通信

这一章是后续理解 Nginx、HTTPS 和 API 的基础。

### 3.1 什么是 IP 地址？

IP 地址是网络层用于标识网络接口、并支持数据包路由的一类地址。

可以暂时把它理解为：网络通信中，数据包要去往的目标地址。

常见的 IP 协议版本有：

- IPv4，例如 `192.168.1.10`
- IPv6，例如 `2001:db8::10`

IPv4 地址通常以四个十进制数字表示，每个数字范围是 0～255。

但要注意：IP 地址并不总是唯一对应一台物理机器。一个服务器可以有多个网络接口和多个 IP，一个 IP 也可能通过 NAT 或负载均衡对应多个后端服务。

### 3.2 公网 IP 与私网 IP

IPv4 私有地址主要包括以下三个范围：

| 地址范围 | 常见用途 |
|---|---|
| `10.0.0.0/8` | 大型内部网络、云 VPC |
| `172.16.0.0/12` | 内部网络 |
| `192.168.0.0/16` | 家庭和办公网络 |

例如：

```text
你的电脑：192.168.1.20
家中路由器的公网 IP：203.0.113.10
云服务器公网 IP：198.51.100.20
云服务器私网 IP：10.0.1.15
```

这里的公网 IP 同样是文档示例地址。

家庭电脑的私网 IP 通常无法直接从互联网访问。家庭路由器通过 NAT（网络地址转换）等机制，让多个内部设备共享公网出口。

云服务器也可能存在类似的网络转换机制，具体取决于云厂商的网络设计。

#### 如何查看 Linux 服务器的 IP？

```bash
ip addr
```

查看路由：

```bash
ip route
```

查看本机网络接口信息：

```bash
ip -br addr
```

注意：`ip addr` 显示的是操作系统网络接口上配置的地址。某些云平台的公网 IP 是通过外部映射提供的，因此它不一定直接显示在系统网卡上。

### 3.3 什么是端口？

IP 地址主要解决“数据包发往哪里”的问题。

端口用于标识某个 IP 地址上的具体传输层服务。

例如：

```text
203.0.113.10:22
203.0.113.10:80
203.0.113.10:443
203.0.113.10:8000
```

它们分别可以表示同一台服务器上的 SSH、HTTP、HTTPS 和某个自定义 Web 服务。

常见端口：

| 端口 | 常见用途 |
|---|---|
| 22 | SSH 远程管理 |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 5432 | PostgreSQL 默认端口 |
| 3306 | MySQL 默认端口 |
| 8000 | 常见的开发服务器端口 |
| 8080 | 常见的备用 HTTP 服务端口 |

端口号本身并不决定程序的性质。比如 8000 端口既可以运行 FastAPI，也可以运行其他 TCP 服务。

端口范围为 0～65535，其中 0 通常不用于普通应用主动绑定服务，1～1023 为传统意义上的知名端口范围。

### 3.4 IP + 端口为什么很重要？

假设一台服务器上运行三个程序：

```text
Nginx：       0.0.0.0:443
FastAPI：     127.0.0.1:8000
内部服务：    127.0.0.1:9000
```

这意味着：

- Nginx 在服务器的所有 IPv4 网络接口上监听 443 端口。
- FastAPI 只监听本机回环地址的 8000 端口。
- 内部服务只监听本机回环地址的 9000 端口。

外部用户通过 HTTPS 访问 Nginx，Nginx 再把请求转发给 FastAPI。

因此，用户不需要直接访问 FastAPI 的 8000 端口。

这就是为什么生产环境中，Web 服务经常采用“统一公网入口 + 内部应用服务”的设计。

### 3.5 TCP 和 UDP 是什么？

IP 负责网络层寻址与路由。传输层则提供不同形式的端到端通信。

#### TCP

TCP 是一种面向连接的、提供可靠有序字节流的传输协议。

它通常会处理：

- 建立连接
- 数据顺序
- 丢失数据的重传
- 流量控制
- 拥塞控制

HTTPS、HTTP/1.1 和 HTTP/2 通常运行在 TCP 之上。

#### UDP

UDP 提供无连接的数据报传输，不保证数据一定到达，也不保证到达顺序。

它常用于 DNS、实时音视频和其他对延迟敏感的通信。

HTTP/3 则基于 QUIC，而 QUIC 使用 UDP 作为底层传输。

所以，不应简单地认为所有 HTTP 通信都必须使用 TCP。

### 3.6 一次浏览器请求实际经过什么？

假设你访问：

```text
https://factor.example.com/api/factors
```

大致流程如下：

1. 浏览器解析 URL，识别协议、域名、路径等信息。
2. 浏览器查询 DNS，获取域名对应的 IP 地址。
3. 浏览器根据目标地址建立相应的传输连接。
4. 如果使用 HTTPS，客户端与服务端执行 TLS 握手。
5. 浏览器发送 HTTP 请求。
6. 请求抵达服务器网络接口。
7. 操作系统网络协议栈处理数据包。
8. 数据交给监听对应端口的服务进程。
9. Nginx 读取 HTTP 请求并决定如何处理。
10. Nginx 将请求代理到 FastAPI。
11. FastAPI 执行业务逻辑，查询 DuckDB 或其他数据源。
12. FastAPI 生成 HTTP 响应。
13. Nginx 将响应返回给客户端。
14. 浏览器解析响应并显示页面或数据。

实际连接可能复用既有连接，也可能通过代理、负载均衡或 CDN，因此上述流程是用于理解的典型模型，而非所有请求都必须完整重复执行的固定步骤。

### 3.7 什么是监听地址？为什么 `127.0.0.1` 很关键？

常见的监听地址包括：

```text
127.0.0.1
0.0.0.0
具体的网络接口 IP
```

`127.0.0.1` 是 IPv4 回环地址，代表本机。

例如：

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

它意味着服务只接受通过该回环接口访问的连接。

因此，同一台服务器上的 Nginx 可以访问：

```text
http://127.0.0.1:8000
```

但外部电脑不能直接通过服务器公网 IP 的 8000 端口访问这个服务。

而：

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

通常表示监听所有 IPv4 网络接口。

**`0.0.0.0` 是服务端的绑定地址，不是客户端应该访问的目标地址。**

如果同时放行公网 8000 端口，这个服务就可能直接暴露在互联网中。

IPv6 还有相应的回环地址 `::1` 和通配地址 `::`。IPv4 与 IPv6 的监听行为应当分别检查，不能假设绑定 IPv4 就自动覆盖 IPv6。

---

## 第四章：为什么不能随意对外暴露端口？

### 4.1 暴露端口究竟意味着什么？

暴露端口意味着外部网络具备了访问某个服务的网络路径。

它不一定意味着任何人都能成功登录或使用服务，但意味着服务的攻击面扩大了。

例如：

```text
公网
 |
 +---- 22：SSH
 |
 +---- 443：Nginx
 |
 +---- 8000：FastAPI
 |
 +---- 5432：PostgreSQL
```

如果这些端口都允许所有来源访问，那么攻击者就可能尝试：

- 扫描端口并识别服务。
- 利用应用程序漏洞。
- 尝试弱密码或泄露的凭据。
- 访问未设置身份验证的 API。
- 利用数据库配置错误获取数据。
- 对服务进行拒绝服务攻击。
- 利用有缺陷的管理接口执行危险操作。

端口扫描本身不等于入侵，但它是攻击者识别目标服务的常见步骤。

### 4.2 为什么 FastAPI 的 8000 端口通常不直接开放？

开发时常用：

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

这种方式很方便，但直接部署到公网可能带来问题：

1. 应用直接承担所有公网请求。
2. HTTPS 终止、证书管理等工作需要额外配置。
3. 缺少统一的反向代理入口。
4. 多个应用服务不容易共用一个公网端口。
5. 访问日志、限流、请求大小限制等配置容易分散。
6. 开发服务器的运行模式不一定适合生产环境。
7. 应用自身的接口可能被意外暴露。

更常见的方式是：

```text
公网 :443
    |
    v
  Nginx
    |
    v
127.0.0.1:8000
    |
    v
 FastAPI
```

此时，公网通常只需要开放 443；如需将 HTTP 自动跳转到 HTTPS，再开放 80。

SSH 管理端口则应该根据管理需求进行限制。

### 4.3 云安全组和 Linux 防火墙有什么区别？

云服务器通常至少涉及两层访问控制。

#### 云厂商安全组

安全组是云平台层面的网络访问控制机制。

它可以控制哪些来源 IP、哪些协议和端口允许进入实例。

例如：

```text
允许：
TCP 443，来源 0.0.0.0/0
TCP 80，来源 0.0.0.0/0
TCP 22，来源为指定管理 IP

拒绝或不允许：
TCP 8000，来自任意公网地址
TCP 5432，来自任意公网地址
```

具体默认规则取决于云厂商。

#### Linux 防火墙

Linux 主机自身还可能通过 nftables、iptables、UFW 或 firewalld 等工具控制流量。

它作用于操作系统网络层面的访问控制。

两者不是同一个东西：

- 安全组控制云平台层面的流量。
- 主机防火墙控制操作系统内部的流量。

一条请求要成功到达服务，必须满足整个网络路径上的相关规则。

即使安全组允许 443，Linux 防火墙仍可能阻止请求；反过来，即使主机防火墙允许 443，云安全组也可能拒绝流量。

### 4.4 如何设计最小暴露面？

推荐遵循最小权限原则。

| 服务 | 推荐暴露方式 |
|---|---|
| SSH | 仅允许必要的管理来源访问；条件允许时优先使用 VPN、堡垒机或云厂商管理通道 |
| Nginx HTTP | 按需要开放 80，通常用于 HTTPS 跳转或证书验证 |
| Nginx HTTPS | 对外开放 443 |
| FastAPI | 只绑定本机回环地址或内部私网地址 |
| DuckDB | 通常作为应用内嵌查询引擎使用，不需要独立公网端口 |
| PostgreSQL / MySQL | 默认只允许本机或受控的内部网络访问 |
| 定时任务 | 一般不需要开放入站端口 |

特别需要注意：**DuckDB 与传统数据库服务器不同，它通常作为进程内分析数据库使用，不需要为了提供查询能力就开放一个数据库网络端口。**

对于你的因子看板项目，FastAPI 可以在内部读取 Parquet 和 DuckDB 文件，外部用户只访问 FastAPI 提供的 HTTP 接口。

---

## 第五章：如何登录和管理云服务器？

### 5.1 SSH 是什么？

SSH（Secure Shell）是一种用于安全远程登录和执行命令的网络协议。

它常用于：

- 远程登录 Linux。
- 执行系统管理命令。
- 传输文件。
- 创建加密隧道。
- 通过公钥认证登录服务器。

SSH 默认使用 TCP 22 端口，但实际部署中可以修改端口。

SSH 连接一般包括：

```text
你的电脑
   |
   | SSH 客户端
   |
   | TCP 连接
   v
云服务器公网 IP:22
   |
   v
SSH 服务端 sshd
   |
   v
身份认证
   |
   v
远程 Shell
```

### 5.2 如何使用 SSH 登录？

假设：

```text
服务器 IP：203.0.113.10
用户名：ubuntu
```

在本地终端执行：

```bash
ssh ubuntu@203.0.113.10
```

如果服务器使用其他用户名，就替换 `ubuntu`。

不同云镜像的默认用户名可能不同，例如 `ubuntu`、`debian`、`ec2-user` 或 `root`。应以云厂商和镜像说明为准。

如果 SSH 使用非默认端口：

```bash
ssh -p 2222 ubuntu@203.0.113.10
```

### 5.3 SSH 密码认证与密钥认证

SSH 常见的身份认证方式包括密码认证和公钥认证。

#### 密码认证

用户输入密码，服务器验证密码是否正确。

这种方式容易理解，但如果密码较弱、被复用或泄露，就可能被攻击者利用。

#### 公钥认证

公钥认证通常使用一对密钥：

- 私钥：由客户端安全保管。
- 公钥：安装在服务器允许登录的用户账户中。

客户端使用私钥完成相应的密码学证明，服务器根据已配置的公钥验证身份。

可以在本地生成密钥：

```bash
ssh-keygen -t ed25519 -C "my-server-key"
```

生成的文件通常包括：

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

私钥不能随意发送给别人，也不应该上传到 Git 仓库。

公钥则可以部署到服务器的授权密钥列表中。

如果云厂商在创建服务器时支持绑定 SSH 公钥，优先使用这种方式。

在确认密钥登录正常、且拥有其他恢复途径后，才考虑关闭密码登录。不要在没有验证新登录方式之前贸然修改 SSH 配置，否则可能把自己锁在服务器之外。

### 5.4 登录后应该先做什么？

先确认操作系统、内核和当前用户：

```bash
whoami
hostname
uname -a
cat /etc/os-release
```

查看 CPU、内存和磁盘：

```bash
lscpu
free -h
df -h
```

查看系统运行状态：

```bash
uptime
top
```

如果安装了 `htop`，也可以使用它查看进程资源消耗。

检查网络接口：

```bash
ip -br addr
ip route
```

这些命令有助于回答：

- 当前使用什么操作系统？
- 我以什么身份登录？
- 服务器有多少内存？
- 磁盘还剩多少空间？
- 哪些进程正在消耗资源？
- 网络接口配置是否正常？

### 5.5 Linux 文件系统应该怎样理解？

Linux 使用统一的目录树组织文件，而不是像 Windows 那样主要通过 C 盘、D 盘区分文件系统。

常见目录：

| 目录 | 用途 |
|---|---|
| `/` | 文件系统根目录 |
| `/home` | 普通用户的主目录通常位于这里 |
| `/root` | root 用户的主目录 |
| `/etc` | 系统与应用配置文件 |
| `/var` | 日志、缓存、状态数据等经常变化的文件 |
| `/var/log` | 常见日志存放位置 |
| `/usr` | 大量系统程序、库和共享资源 |
| `/tmp` | 临时文件 |
| `/opt` | 常用于安装额外的软件 |
| `/srv` | 可用于存放服务提供的数据 |
| `/mnt` | 常用于临时挂载文件系统 |
| `/proc` | 内核和进程相关的虚拟文件系统 |
| `/dev` | 设备文件 |

你的项目可以采用类似结构：

```text
/opt/crypto-factor/
├── app/
│   ├── main.py
│   ├── api/
│   └── services/
├── scripts/
│   ├── collect_market_data.py
│   └── build_factors.py
├── config/
├── pyproject.toml
└── README.md

/srv/crypto-factor-data/
├── raw/
│   └── exchange=binance/
├── factors/
├── metadata/
└── backups/

/var/log/crypto-factor/
```

这是一种示例布局，不是 Linux 强制要求。

程序代码、数据文件、日志和配置分开放置，后续升级程序或备份数据时更容易管理。

### 5.6 Linux 常用命令

#### 查看文件

```bash
ls -lah
pwd
find . -maxdepth 2 -type f
```

#### 创建目录

```bash
mkdir -p /opt/crypto-factor
```

#### 复制和移动文件

```bash
cp source.txt destination.txt
mv old_name.txt new_name.txt
```

#### 查看文件内容

```bash
cat config.yaml
less application.log
tail -n 100 application.log
tail -f application.log
```

`tail -f` 常用于持续观察日志追加内容。

#### 查找进程

```bash
ps aux
pgrep -af uvicorn
```

#### 终止进程

```bash
kill PID
```

应优先使用应用自身的关闭方式或服务管理器停止程序。只有在确认目标进程及其影响之后，才考虑强制终止。

#### 查看磁盘占用

```bash
du -sh /srv/crypto-factor-data
df -h
```

`du` 主要查看目录或文件占用，`df` 查看文件系统空间使用情况。

### 5.7 Linux 权限与 sudo

Linux 文件通常具有所有者、所属组和其他用户三类权限控制。

常见权限：

- `r`：读取
- `w`：写入
- `x`：执行，或在目录上表示可进入和访问目录项

查看权限：

```bash
ls -l
```

例如：

```text
-rw-r----- 1 appuser appuser 1024 Oct 18 config.yaml
```

表示文件所有者可以读写，所属组可以读取，其他用户没有权限。

`sudo` 用于在授权范围内以更高权限执行命令。

例如：

```bash
sudo systemctl status nginx
```

不要习惯性地使用 root 运行所有 Python 程序。更好的做法是为应用创建专用系统用户，只授予它所需的目录和文件权限。

对于你的行情采集和因子研究程序，可以让应用用户拥有数据目录的写入权限，而不必拥有修改整个系统的权限。

---

## 第六章：Linux 上的进程、服务与资源管理

### 6.1 什么是进程？

进程是操作系统对正在运行的程序实例进行管理的基本单位之一。

当你执行：

```bash
python scripts/collect_market_data.py
```

操作系统会启动 Python 解释器进程，加载脚本并执行。

一个进程通常具有：

- 进程 ID（PID）
- 内存空间
- 打开的文件描述符
- 环境变量
- 用户与权限
- 当前工作目录
- 线程等执行资源

一个程序可以启动多个进程，一个进程也可以包含多个线程。

不要把“程序文件”和“正在运行的程序实例”混为一谈。

`main.py` 是磁盘上的文件，而运行它的 Python 进程是操作系统管理的执行实体。

### 6.2 什么是前台进程和后台进程？

直接执行：

```bash
python scripts/collect_market_data.py
```

程序通常会占据当前终端的前台。

终端显示程序输出，程序也可以接收终端输入。

如果你关闭终端或 SSH 连接，进程是否继续运行会受到会话、终端和信号处理等因素影响。不能把普通终端运行方式视为可靠的长期运行机制。

临时后台执行可以使用：

```bash
python scripts/collect_market_data.py > collector.log 2>&1 &
```

其中：

- `>` 将标准输出重定向到日志文件。
- `2>&1` 将标准错误也重定向到同一位置。
- `&` 让 Shell 将命令放入后台执行。

但这只是一个简单的后台运行方式。

它通常不具备完整的服务管理能力，例如：

- 自动重启。
- 开机启动。
- 统一状态查询。
- 明确的服务生命周期管理。
- 完整的运行依赖和权限管理。

生产环境应使用服务管理器或专门的进程管理工具。

### 6.3 什么是守护进程？

守护进程（Daemon）通常指在后台长期运行、为系统或其他程序提供服务的进程。

例如：

- SSH 服务进程。
- Nginx 工作进程。
- 某些数据库服务进程。
- 长期运行的数据采集服务。

不过，现代 Linux 中的长期服务不一定需要程序自己执行传统意义上的“守护化”操作。程序可以直接以前台方式运行，由 systemd 等服务管理器负责其生命周期。

这种方式通常更容易管理。

---

## 第七章：systemd——让 Python 程序稳定长期运行

### 7.1 为什么需要 systemd？

假设你的行情采集程序每天都要运行，或者有一个持续运行的 Web API。

你需要回答：

- 程序是否正在运行？
- 程序崩溃之后怎么办？
- 服务器重启后怎么办？
- 如何查看程序日志？
- 如何启动、停止和重启程序？
- 程序应该使用什么用户运行？
- 如何确保程序从正确的工作目录启动？

systemd 是许多主流 Linux 发行版使用的系统与服务管理组件。

它可以管理服务的启动顺序、运行状态、重启策略、依赖关系、日志以及开机启动等。

### 7.2 先建立 Python 项目运行环境

假设项目位于：

```text
/opt/crypto-factor
```

创建虚拟环境：

```bash
cd /opt/crypto-factor
python3 -m venv .venv
```

安装依赖：

```bash
/opt/crypto-factor/.venv/bin/pip install fastapi uvicorn duckdb pyarrow pandas ccxt
```

实际项目应根据锁定的依赖清单安装，不应把这条示例命令视为完整的生产依赖管理方案。

检查解释器：

```bash
/opt/crypto-factor/.venv/bin/python --version
```

在服务配置中直接使用虚拟环境内的解释器，可以避免系统 Python、其他虚拟环境和 PATH 配置造成的歧义。

### 7.3 创建 systemd 服务文件

例如，为 FastAPI 创建：

```text
/etc/systemd/system/crypto-factor-api.service
```

配置内容：

```ini
[Unit]
Description=Crypto Factor FastAPI Service
After=network.target

[Service]
Type=simple

User=appuser
Group=appuser

WorkingDirectory=/opt/crypto-factor

EnvironmentFile=-/etc/crypto-factor/api.env

ExecStart=/opt/crypto-factor/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

说明：

- `Description`：服务描述。
- `Type=simple`：将启动命令运行的进程作为主要服务进程管理。
- `User`、`Group`：指定运行用户和用户组。
- `WorkingDirectory`：指定工作目录。
- `EnvironmentFile`：可选的环境变量文件。
- `ExecStart`：实际启动命令。
- `Restart=on-failure`：发生符合条件的失败时尝试重启。
- `RestartSec=5`：重启前等待五秒。
- `WantedBy=multi-user.target`：允许服务配置为随系统正常启动。

这里假设 `appuser` 已经创建，项目目录及数据目录的权限也已正确配置。

`After=network.target` 只规定启动顺序，不保证 DNS、外网访问或交易所 API 已经完全可用。因此，采集程序本身仍然需要处理网络失败、超时和重试。

### 7.4 启用和管理服务

修改 unit 文件之后，先让 systemd 重新加载配置：

```bash
sudo systemctl daemon-reload
```

启动服务：

```bash
sudo systemctl start crypto-factor-api
```

设置开机启动：

```bash
sudo systemctl enable crypto-factor-api
```

立即启动并设置开机启动：

```bash
sudo systemctl enable --now crypto-factor-api
```

查看状态：

```bash
sudo systemctl status crypto-factor-api
```

重启：

```bash
sudo systemctl restart crypto-factor-api
```

停止：

```bash
sudo systemctl stop crypto-factor-api
```

查看日志：

```bash
sudo journalctl -u crypto-factor-api
```

持续观察日志：

```bash
sudo journalctl -u crypto-factor-api -f
```

查看最近一小时的日志：

```bash
sudo journalctl -u crypto-factor-api --since "1 hour ago"
```

这些命令构成了日常管理 Python 服务的基础。

### 7.5 systemd 的重启机制不等于业务可靠性

即使设置了：

```ini
Restart=on-failure
```

也不意味着程序一定能够正确工作。

例如，行情采集程序可能出现以下情况：

- 进程仍然存在，但已经无法访问交易所。
- API 请求持续超时。
- 任务不断重试，导致请求频率过高。
- 程序运行正常，但写入了不完整的 Parquet 文件。
- 服务没有崩溃，却一直返回错误数据。

因此，可靠性至少包含三层：

1. **进程可靠性：** 进程崩溃后能够恢复。
2. **业务可靠性：** 请求失败、数据异常和重复执行得到正确处理。
3. **可观测性：** 可以发现服务是否异常、数据是否过期、任务是否成功。

对量化数据管道而言，后两层尤其重要。

---

## 第八章：cron——让程序每天自动执行

### 8.1 cron 是什么？

cron 是传统 Unix/Linux 定时任务调度机制。

它适用于：

- 每天固定时间运行行情采集任务。
- 每小时更新数据。
- 每天计算因子。
- 定期清理日志。
- 定期生成研究报告。
- 定期执行备份。

如果一个任务只需要在特定时间运行一次或周期性运行，通常没有必要让它一直占用一个 Python 进程。

这就是定时任务与长期服务的区别。

| 类型 | 典型例子 | 常用管理方式 |
|---|---|---|
| 长期服务 | FastAPI、持续监听的 WebSocket 客户端 | systemd |
| 定时任务 | 每天采集日线行情 | cron 或 systemd timer |
| 临时任务 | 手动执行一次因子计算 | Shell 命令或任务运行器 |

### 8.2 cron 的基本语法

典型格式：

```text
分钟 小时 日 月 星期 命令
```

例如：

```cron
0 2 * * * /opt/crypto-factor/scripts/run_daily.sh
```

含义是：每天凌晨 2 点执行脚本。

再看几个例子：

```cron
*/15 * * * * /opt/crypto-factor/scripts/update_data.sh
```

每 15 分钟执行一次。

```cron
0 * * * * /opt/crypto-factor/scripts/hourly_job.sh
```

每小时的第 0 分钟执行一次。

```cron
30 3 * * 1 /opt/crypto-factor/scripts/weekly_job.sh
```

每周一凌晨 3:30 执行。

cron 的时区和日期语义需要结合具体实现确认。云服务器时区可能与本地电脑不同，应该明确任务采用的时区，避免把 UTC 与本地时间混淆。

### 8.3 配置定时任务

编辑当前用户的定时任务：

```bash
crontab -e
```

查看：

```bash
crontab -l
```

例如：

```cron
0 2 * * * /opt/crypto-factor/scripts/run_daily.sh >> /var/log/crypto-factor/daily.log 2>&1
```

注意：cron 运行任务时通常不会继承你交互式 Shell 的全部环境变量和 PATH。

因此，脚本应该尽可能使用绝对路径。

### 8.4 为什么推荐使用 Shell 脚本包装 Python 任务？

创建：

```text
/opt/crypto-factor/scripts/run_daily.sh
```

内容：

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

PROJECT_DIR="/opt/crypto-factor"
PYTHON="$PROJECT_DIR/.venv/bin/python"

cd "$PROJECT_DIR"

"$PYTHON" scripts/collect_market_data.py
"$PYTHON" scripts/build_factors.py
```

赋予执行权限：

```bash
chmod +x /opt/crypto-factor/scripts/run_daily.sh
```

这个脚本做了几件事：

- 固定项目目录。
- 固定 Python 解释器。
- 确保后续命令在正确目录运行。
- 让前一个任务失败时不再盲目继续执行后面的任务。

但还需要注意：如果采集任务因为异常退出，因子任务是否应该继续运行，取决于你的业务设计。对于依赖最新行情的因子计算，通常应先确认数据更新成功。

### 8.5 定时任务需要防止重复执行

假设每天凌晨的行情采集任务需要 20 分钟，但下一次调度已经开始。

如果两个任务同时运行，可能出现：

- 重复请求。
- 同时写入相同的 Parquet 分区。
- 文件覆盖。
- 数据竞争。
- 计算结果不一致。

一种简单的保护方法是使用 `flock`：

```cron
0 2 * * * flock -n /var/lock/crypto-factor-daily.lock /opt/crypto-factor/scripts/run_daily.sh >> /var/log/crypto-factor/daily.log 2>&1
```

如果锁已经被占用，新任务就不会获得锁并开始运行。

实际使用时需要确认系统安装了 `flock`，且锁文件所在目录对运行用户可访问。

此外，锁只能解决特定范围内的并发问题，不能替代程序自身的数据一致性设计。

### 8.6 如何让任务具备可重试能力？

例如，采集交易所日线行情时，可能发生网络超时。

不应简单地让程序无限重试。

更合理的策略包括：

- 为 HTTP 请求设置超时。
- 对临时网络错误进行有限次数重试。
- 使用指数退避，避免连续高频请求。
- 遵守交易所的速率限制。
- 将失败任务记录到日志。
- 支持从上次成功位置继续采集。
- 对重复获取的数据进行幂等处理。
- 只有在数据校验通过后才标记任务成功。

这些是数据管道的工程要求，不是 cron 本身提供的功能。

---

## 第九章：Nginx 是什么？为什么需要它？

### 9.1 Nginx 的基本定位

Nginx 是一种 Web 服务器，也经常被用作反向代理服务器、负载均衡器和静态资源服务器。

在你的 Python 项目中，它通常位于 FastAPI 前面。

```text
浏览器
   |
   | HTTPS :443
   v
 Nginx
   |
   | HTTP :8000
   v
 FastAPI
   |
   v
 DuckDB / Parquet
```

Nginx 并不取代 FastAPI。

两者的职责不同：

- Nginx 负责网络入口、TLS、代理、静态资源和部分流量管理。
- FastAPI 负责业务逻辑、数据查询和 API 响应。

### 9.2 正向代理与反向代理

这是理解 Nginx 的关键概念。

#### 正向代理

正向代理站在客户端一侧。

```text
客户端
   |
   v
正向代理
   |
   v
目标服务器
```

客户端通过代理访问目标服务。

目标服务器看到的连接来源通常是代理服务器，而不是原始客户端的直接连接，但实际地址传递行为取决于具体协议和代理配置。

#### 反向代理

反向代理站在服务端一侧。

```text
客户端
   |
   v
反向代理
   |
   +------> 应用 A
   |
   +------> 应用 B
   |
   +------> 静态文件
```

客户端访问统一入口，不必知道内部服务的具体地址。

例如：

```text
https://factor.example.com/
```

可能由 Nginx 提供网页静态资源。

而：

```text
https://factor.example.com/api/factors
```

则由 Nginx 转发给 FastAPI。

### 9.3 Nginx 为什么不直接代替 FastAPI？

因为两者是不同层次的组件。

FastAPI 适合处理：

- 请求参数校验。
- 数据查询。
- 用户身份与权限逻辑。
- 因子结果计算与序列化。
- 业务异常处理。

Nginx 适合处理：

- HTTPS 连接入口。
- 静态文件。
- 请求转发。
- 基础请求限制。
- 连接与超时配置。
- 多个后端服务之间的路由。

如果把所有网络入口和业务逻辑都混在应用中，会让部署、扩展和安全控制更复杂。

### 9.4 Nginx 如何工作？

Nginx 接收到请求之后，大致会执行以下流程：

1. 接受网络连接。
2. 读取 HTTP 请求。
3. 根据监听地址、域名和路径等信息选择处理规则。
4. 如果请求对应静态文件，就读取并返回文件。
5. 如果请求需要代理，就建立或复用到后端的连接。
6. 把请求转发给后端应用。
7. 读取后端响应。
8. 将响应返回客户端。

Nginx 使用事件驱动的处理架构，适合处理大量网络连接。具体性能取决于配置、负载、硬件和应用后端，不能简单认为引入 Nginx 就必然让所有程序变快。

### 9.5 安装 Nginx

在 Ubuntu 或 Debian 系统上，常见方式是：

```bash
sudo apt update
sudo apt install nginx
```

启动：

```bash
sudo systemctl enable --now nginx
```

检查状态：

```bash
sudo systemctl status nginx
```

测试配置：

```bash
sudo nginx -t
```

重新加载配置：

```bash
sudo systemctl reload nginx
```

如果修改配置后没有执行 reload 或 restart，新配置可能尚未生效。

### 9.6 Nginx 配置文件在哪里？

常见路径包括：

```text
/etc/nginx/nginx.conf
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
/etc/nginx/conf.d/
```

不同发行版的布局可能有所不同。

在 Debian/Ubuntu 常见的布局中：

- `nginx.conf` 是主配置文件。
- `sites-available` 存放可用站点配置。
- `sites-enabled` 中的配置通常会被主配置引用。

修改前最好先确认当前系统实际使用的配置路径。

### 9.7 为 FastAPI 配置反向代理

假设：

- 域名为 `factor.example.com`。
- FastAPI 监听 `127.0.0.1:8000`。
- TLS 证书已配置完成。

可以创建类似配置：

```nginx
server {
    listen 80;
    server_name factor.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name factor.example.com;

    ssl_certificate
        /etc/letsencrypt/live/factor.example.com/fullchain.pem;

    ssl_certificate_key
        /etc/letsencrypt/live/factor.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

这里的域名和证书路径都是示例。实际使用时必须替换成自己的域名，并确保文件确实存在。

配置检查：

```bash
sudo nginx -t
```

配置正确后：

```bash
sudo systemctl reload nginx
```

#### 为什么要设置这些请求头？

`Host` 用于传递原始请求中的主机信息。

`X-Real-IP` 和 `X-Forwarded-For` 用于传递代理链中的客户端地址信息。

`X-Forwarded-Proto` 用于传递原始请求的协议，例如 HTTP 或 HTTPS。

这些头本身并不是天然可信的身份凭据。FastAPI 应只信任来自自己控制的代理链的转发信息，并正确配置可信代理范围。否则，客户端可能伪造转发头。

### 9.8 Nginx 的常见故障

#### 502 Bad Gateway

通常表示 Nginx 无法从后端获得有效响应。

常见原因：

- FastAPI 没有启动。
- FastAPI 监听地址错误。
- 后端端口错误。
- 连接被拒绝。
- 后端进程崩溃。
- 请求超时。

检查：

```bash
sudo systemctl status crypto-factor-api
sudo journalctl -u crypto-factor-api -n 100
sudo tail -n 100 /var/log/nginx/error.log
```

然后在服务器内部直接测试：

```bash
curl -i http://127.0.0.1:8000/
```

#### 404 Not Found

可能是：

- FastAPI 没有对应路由。
- Nginx 的 `location` 规则与路径不匹配。
- `proxy_pass` 的路径处理方式不符合预期。

特别注意，`proxy_pass` 是否带有尾部斜杠，会影响某些配置下的 URI 转发行为。

#### 连接超时

可能是：

- 后端处理时间太长。
- 网络路径有问题。
- 防火墙阻止连接。
- 应用没有及时返回响应。

不要仅仅通过无限增大超时时间掩盖问题，应先判断到底是网络故障还是业务计算耗时过长。

---

## 第十章：域名是什么？DNS 如何工作？

### 10.1 域名与 IP 的关系

IP 地址适合计算机进行网络寻址，但不适合人类记忆。

域名提供了更容易识别和管理的名称。

例如：

```text
factor.example.com
```

通过 DNS 解析，可以获得这个域名对应的 IP 地址或其他 DNS 记录。

但域名不是 IP 地址的另一种写法。

两者是不同层次的标识：

- 域名是 DNS 名称系统中的名称。
- IP 地址是网络层使用的地址。
- DNS 负责将域名解析成相应记录。

域名解析结果可以随时间变化。一个域名也可以对应多个地址，或者通过 CDN、负载均衡等系统指向服务。

### 10.2 DNS 是什么？

DNS（Domain Name System，域名系统）是一套分布式命名系统。

当用户访问：

```text
https://factor.example.com/api/factors
```

浏览器需要知道 `factor.example.com` 应该连接到哪里。

通常，它会通过操作系统解析器、递归 DNS 解析器等获取记录。

解析过程可能涉及：

1. 检查本地缓存。
2. 查询递归解析器。
3. 递归解析器查询根 DNS。
4. 查询顶级域名服务器。
5. 查询权威 DNS 服务器。
6. 获取对应的记录并缓存。

实际解析中可能使用缓存直接获得结果，因此并不是每次访问都从根服务器重新开始查询。

### 10.3 常见 DNS 记录

| 记录 | 作用 |
|---|---|
| A | 将域名映射到 IPv4 地址 |
| AAAA | 将域名映射到 IPv6 地址 |
| CNAME | 将一个名称指向另一个规范名称 |
| MX | 指定邮件交换服务器 |
| TXT | 存放文本记录，常用于域名验证等场景 |
| NS | 指定域名的权威 DNS 服务器 |

例如：

```text
factor.example.com  A  198.51.100.20
```

表示该名称对应一个 IPv4 地址。

如果云服务器的公网 IP 发生变化，而 DNS 记录仍然指向旧地址，用户就可能访问错误的服务器。

### 10.4 DNS 记录修改为什么不会总是立即生效？

DNS 记录通常带有 TTL（Time To Live，生存时间）。

递归解析器可以在 TTL 允许的时间范围内缓存结果。

因此，修改记录之后：

- 有些用户可能立即看到新结果。
- 有些用户可能继续使用缓存的旧结果。
- 具体生效时间受到 TTL、缓存行为和解析链路影响。

如果服务器公网 IP 会变化，可以考虑使用固定公网 IP 或经过设计的动态 DNS 方案。

### 10.5 域名与 URL 有什么区别？

例如：

```text
https://factor.example.com/api/factors?limit=100
```

它是一个 URL（统一资源定位符）。

拆解如下：

| 部分 | 内容 | 含义 |
|---|---|---|
| 协议方案 | `https` | 使用 HTTPS |
| 主机名 | `factor.example.com` | 目标服务名称 |
| 路径 | `/api/factors` | 请求的资源或接口路径 |
| 查询参数 | `limit=100` | 附加请求参数 |

如果 URL 显式包含端口，例如：

```text
https://factor.example.com:8443/api/factors
```

那么连接会使用 8443 端口，而不是 HTTPS 的默认端口 443。

URL 中还可以出现用户名、密码片段或其他部分，但生产环境不应把秘密凭据放在容易泄露的 URL 中。

---

## 第十一章：HTTPS、TLS 与证书

### 11.1 HTTP 是什么？

HTTP（Hypertext Transfer Protocol，超文本传输协议）是一种应用层协议。

它定义了客户端如何发送请求、服务端如何返回响应。

例如：

```http
GET /api/factors HTTP/1.1
Host: factor.example.com
Accept: application/json
```

这表示客户端希望获取 `/api/factors` 对应的资源。

服务端可能返回：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"count": 100}
```

上面只是示意响应。

HTTP 本身并不提供传输内容的加密保护。若使用明文 HTTP，网络路径上的观察者在能够读取相关流量的情况下，可能看到请求内容。

### 11.2 HTTPS 是什么？

HTTPS 可以理解为在 HTTP 通信中加入 TLS 安全层。

TLS 提供的主要能力包括：

- 机密性：加密通信内容。
- 完整性：检测传输内容是否被篡改。
- 服务端身份认证：客户端可以验证自己连接的服务端是否符合证书和域名验证规则。

如果配置了双向 TLS，还可以让服务端验证客户端证书。

HTTPS 并不意味着整个系统自动安全。

例如：

- API 没有身份认证，攻击者仍可能调用它。
- 服务端存在漏洞，攻击者仍可能利用漏洞。
- 用户在钓鱼网站输入密码，TLS 也可能正常工作。
- 数据在服务器内部仍可能以明文形式存在。

HTTPS 保护的是通信链路的重要部分，而不是替代所有安全措施。

### 11.3 TLS 握手的基本过程

简化理解：

```text
客户端
   |
   | ClientHello
   v
服务端
   |
   | ServerHello + 证书等握手信息
   v
客户端验证证书及域名
   |
   | 协商密钥与相关握手消息
   v
双方建立受保护的通信
   |
   v
传输加密的 HTTP 数据
```

实际 TLS 1.3 握手包含更多细节，上述图示省略了部分消息。

关键点是：

1. 双方协商协议版本和密码套件等参数。
2. 服务端提供证书链等身份验证材料。
3. 客户端验证证书是否可信、是否匹配访问的主机名、是否在有效期内等。
4. 双方通过密钥交换建立会话密钥。
5. 后续应用数据通过协商出的密钥进行加密和完整性保护。

TLS 不一定通过非对称加密直接加密所有 HTTP 数据。现代 TLS 通常通过密钥交换建立会话密钥，再使用对称加密保护大量应用数据。

### 11.4 什么是数字证书？

数字证书是一种由证书颁发机构（CA）签发的数字文件。

它包含证书持有者的身份相关信息、公开密钥、有效期、用途等信息，并由签发机构进行数字签名。

浏览器通过信任的根证书和证书链验证服务端证书。

如果证书有效、域名匹配、签发链可信且其他验证条件满足，客户端就可以在相应的安全模型下确认服务端身份。

#### 证书与域名有什么关系？

假设你访问：

```text
https://factor.example.com
```

那么证书应当适用于 `factor.example.com`。

如果证书只适用于 `api.example.com`，直接访问 `factor.example.com` 时通常会发生主机名验证失败。

因此，申请证书之前，必须明确要保护哪些域名。

### 11.5 什么是 Let's Encrypt？

Let's Encrypt 是提供免费 TLS 证书的证书颁发机构。

它支持通过自动化协议完成域名控制权验证和证书签发。

常见工具包括 Certbot。

典型流程：

1. 购买或准备域名。
2. 将 DNS 记录指向服务器。
3. 配置必要的网络访问规则。
4. 申请证书。
5. 配置 Nginx 使用证书。
6. 测试 HTTPS 是否正常。
7. 设置并验证证书自动续期。

免费证书并不意味着不需要维护。证书有有效期，需要正确续期。

### 11.6 为什么生产环境要自动续期？

如果证书过期，客户端可能拒绝连接，或者显示明显的安全警告。

这会导致用户无法正常访问网站。

因此，证书自动续期应成为部署的一部分。

使用 Certbot 时，可以根据安装方式检查续期任务是否已经配置：

```bash
sudo certbot renew --dry-run
```

该命令通常用于测试续期流程，具体行为取决于 Certbot 的安装和配置。

不要假设安装 Certbot 后，所有证书续期机制一定已经正确运行。

### 11.7 HTTP、HTTPS 与端口

常见约定：

```text
HTTP  -> TCP 80
HTTPS -> TCP 443
```

但端口号本身不是协议的强制属性。

HTTP 可以运行在其他端口，HTTPS 也可以运行在其他端口。

如果用户访问：

```text
https://factor.example.com
```

浏览器默认使用 443。

如果访问：

```text
http://factor.example.com
```

浏览器默认使用 80。

一种常见生产部署方式是：

```text
80：接收 HTTP 并跳转到 HTTPS
443：处理 HTTPS 请求
8000：仅供内部 FastAPI 使用
```

---

## 第十二章：API 的本质与工作原理

这一章不仅涉及 FastAPI，还会帮助你理解交易所 API、前端接口和服务之间的通信。

### 12.1 API 是什么？

API 是 Application Programming Interface，即应用程序编程接口。

从概念上讲，API 是一个系统向其他程序提供的、约定明确的交互方式。

API 不一定是网络服务。

例如：

```python
import math

result = math.sqrt(16)
```

这里调用 `math.sqrt()` 就是一种程序接口调用。

它不需要网络，也不一定涉及 HTTP。

另一个例子：

```python
def calculate_factor(data):
    ...
```

函数的参数、返回值和错误约定，同样构成程序接口的一部分。

但在服务器部署场景中，我们经常讨论的是 **Web API**：通过 HTTP 等网络协议向其他程序提供服务的接口。

### 12.2 Web API 是什么？

Web API 通过网络请求与响应，让不同程序交换数据或执行操作。

例如：

```text
浏览器
   |
   | HTTP GET /api/factors
   v
FastAPI
   |
   | 查询 DuckDB
   v
Parquet 数据
   |
   v
FastAPI
   |
   | HTTP JSON 响应
   v
浏览器
```

浏览器不需要知道 Parquet 的文件路径，也不需要直接操作 DuckDB。

它只需要按照约定访问 API。

这种设计把内部数据处理与外部使用者解耦。

### 12.3 API 为什么需要明确的契约？

假设你的 API 返回：

```json
{
  "symbol": "BTC/USDT",
  "date": "2026-10-17",
  "factor_name": "momentum_20d",
  "factor_value": 0.032
}
```

调用者需要知道：

- `symbol` 表示什么？
- `date` 使用哪个时区？
- `factor_name` 是否有固定的命名规范？
- `factor_value` 是原始值还是标准化值？
- 数据缺失时返回什么？
- 请求失败时会返回什么状态码？

如果这些约定不明确，即使 API 可以正常返回 JSON，调用者也可能错误地理解数据。

所以，API 不只是一个 URL，它还包括请求格式、响应格式、认证、错误处理、版本兼容和语义约定等。

### 12.4 REST API 是什么？

REST 是一种软件架构风格。RESTful API 通常围绕资源组织接口，并利用 HTTP 的语义。

例如：

```text
GET    /api/factors
GET    /api/factors/momentum_20d
GET    /api/factors/momentum_20d/values
POST   /api/research-runs
DELETE /api/research-runs/123
```

这些路径只是示例，具体接口应由实际业务设计决定。

常见 HTTP 方法：

| 方法 | 典型语义 |
|---|---|
| GET | 获取资源 |
| POST | 提交数据、创建资源或执行某些操作 |
| PUT | 整体替换资源 |
| PATCH | 部分更新资源 |
| DELETE | 删除资源 |

这些方法有各自的标准语义。实际 API 设计应尽量保持一致，避免用 GET 执行会改变重要业务状态的操作。

### 12.5 GET 请求示例

例如：

```http
GET /api/factors?symbol=BTC%2FUSDT&limit=100 HTTP/1.1
Host: factor.example.com
Accept: application/json
```

查询参数可以用于：

- 选择交易对。
- 指定因子。
- 限定日期范围。
- 控制返回行数。
- 选择排序方式。

但服务端必须验证参数，不能因为客户端传入了 `limit=100000000` 就无条件读取海量数据。

### 12.6 POST 请求示例

例如，提交一个研究任务：

```http
POST /api/research-runs HTTP/1.1
Host: factor.example.com
Content-Type: application/json

{
  "factor_name": "momentum_20d",
  "start_date": "2025-01-01",
  "end_date": "2026-01-01"
}
```

FastAPI 可以接收并校验请求体，然后创建研究任务。

但需要注意：创建任务与执行任务并不是同一回事。

如果因子计算可能持续数分钟，直接在 HTTP 请求中同步执行，可能导致请求超时或占用 Web 服务资源。

更合理的设计可以是：

1. API 接收任务请求。
2. 校验参数。
3. 创建任务记录。
4. 将任务交给后台任务执行器。
5. 立即返回任务 ID。
6. 客户端通过另一个 API 查询执行状态。

对于第一版小型项目，可以先使用简单的任务表和独立 Python 进程；等任务规模增长后，再考虑更完整的任务队列。

### 12.7 HTTP 状态码是什么？

状态码表示服务器对请求的处理结果类别。

| 状态码 | 含义 |
|---|---|
| 200 | 请求成功 |
| 201 | 资源已创建 |
| 204 | 请求成功，没有响应体 |
| 301 / 302 | 重定向 |
| 400 | 请求错误 |
| 401 | 未通过身份认证 |
| 403 | 已识别身份或请求，但不允许访问 |
| 404 | 资源不存在或接口路径不匹配 |
| 409 | 请求与当前资源状态冲突 |
| 422 | 请求数据未通过验证等处理错误 |
| 429 | 请求过于频繁 |
| 500 | 服务端发生未预期错误 |
| 502 | 网关从上游获得了无效响应 |
| 503 | 服务暂时不可用 |
| 504 | 网关等待上游响应超时 |

具体状态码是否适用，要结合接口语义和框架行为。

例如，FastAPI 在请求参数验证失败时，常见情况下会返回 422。

### 12.8 JSON 是什么？

JSON 是一种轻量级数据交换格式。

例如：

```json
{
  "symbol": "BTC/USDT",
  "close": 67000.5,
  "volume": 12345.6,
  "valid": true,
  "tags": ["daily", "spot"]
}
```

JSON 具有对象、数组、字符串、数字、布尔值和 null 等数据类型。

它便于不同编程语言之间交换数据。

但 JSON 不是 API 本身，也不是 HTTP 本身。

三者的关系是：

- HTTP 是通信协议。
- Web API 定义服务如何被调用。
- JSON 是常见的数据序列化格式。

API 也可以使用 XML、二进制格式、Protocol Buffers 或其他数据表示形式。

### 12.9 FastAPI 如何把 Python 函数变成 HTTP 接口？

示例：

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/api/health")
def health():
    return {"status": "ok"}
```

这段代码定义了一个 GET 路由。

当客户端请求：

```text
GET /api/health
```

FastAPI 会匹配对应路由、执行函数，并将返回的 Python 字典序列化为 JSON 响应。

启动服务：

```bash
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

这里：

- `app.main` 表示 Python 模块路径。
- `app` 表示模块中的 FastAPI 实例。
- `--host` 指定监听地址。
- `--port` 指定监听端口。

这条命令的前提是项目目录结构与模块路径相符。

### 12.10 FastAPI 为什么还需要 Uvicorn？

FastAPI 是基于 ASGI 规范的 Web 框架。

Uvicorn 是 ASGI 服务器，它负责监听网络连接、处理相应的协议交互，并将请求交给 ASGI 应用。

可以理解为：

```text
TCP / HTTP 连接
       |
       v
     Uvicorn
       |
       | ASGI 调用
       v
     FastAPI
       |
       v
     业务逻辑
```

FastAPI 定义应用的路由和业务处理方式；Uvicorn 提供实际的服务运行环境。

因此：

```python
app = FastAPI()
```

并不意味着系统已经自动在公网端口上启动了一个 HTTP 服务。

需要有 ASGI 服务器或其他兼容的运行环境来承载它。

### 12.11 同步与异步 API

FastAPI 支持普通 `def` 和 `async def` 路由。

普通函数：

```python
@app.get("/sync")
def sync_endpoint():
    return {"mode": "sync"}
```

异步函数：

```python
@app.get("/async")
async def async_endpoint():
    return {"mode": "async"}
```

异步的优势主要体现在适合使用异步 I/O 的场景，例如并发等待网络响应。

但 `async def` 不会自动让 CPU 密集型任务变快。

例如，大规模因子计算如果占用大量 CPU，那么将函数写成异步函数并不会自动提高计算速度。

如果在异步路由中直接执行长时间阻塞操作，还可能影响事件循环的响应能力。

对你的研究看板而言，应该区分：

- 快速 API 查询。
- 网络 I/O 型任务。
- 大型数据扫描。
- CPU 密集型因子计算。
- 长时间运行的研究任务。

不同任务应使用适当的执行方式，而不是把所有计算都塞进一个 HTTP 请求。

### 12.12 API 文档为什么重要？

FastAPI 通常会根据路由和类型声明生成 OpenAPI 文档，并提供交互式 API 文档页面。

常见地址：

```text
/docs
/openapi.json
```

这些路径取决于 FastAPI 的配置。

开发环境可以使用交互式文档测试接口。

生产环境则应根据安全需求决定是否公开文档。对于只面向内部使用的管理 API，不应因为框架默认提供文档，就无条件将其暴露给所有互联网用户。

---

## 第十三章：交易所 API 的原理与安全使用

### 13.1 交易所 API 与你自己的 FastAPI 有什么区别？

两者本质上都可能是 Web API，但它们面向不同的服务对象。

交易所 API 通常提供：

- 行情数据。
- 交易对信息。
- 历史 K 线。
- 账户余额。
- 订单创建与查询。
- 持仓查询。
- 资金费率等衍生品数据。

你的 FastAPI 则可能提供：

- 因子列表。
- 因子历史值。
- 分组收益。
- IC / ICIR 统计。
- 数据更新状态。
- 研究任务状态。

它们之间的关系可以是：

```text
交易所 API
    |
    v
Python 数据采集程序
    |
    v
Parquet / DuckDB
    |
    v
你的 FastAPI
    |
    v
因子看板前端
```

### 13.2 API Key 是什么？

某些 API 需要 API Key 来识别调用者，或者配合签名验证请求。

但 API Key 不是所有 API 的必要组成部分。

例如，某些交易所公开市场数据接口可以不登录就访问，而账户、交易和其他敏感接口通常需要认证。

对于私有 API，认证方式可能包括：

- API Key。
- 请求签名。
- 时间戳。
- 随机数或其他防重放机制。
- 额外的请求参数。

具体规则取决于交易所。

### 13.3 为什么交易所 API Key 需要权限管理？

假设你只需要下载公开 K 线数据。

如果创建了具有交易权限的 API Key，即使程序只想读取行情，也扩大了凭据泄露后的风险。

更合理的原则是：

1. 能使用公开接口，就先使用公开接口。
2. 必须认证时，只开启所需权限。
3. 不需要提现权限时，不要开启提现权限。
4. 尽可能限制 API Key 的来源 IP。
5. 不把 API Key 和 Secret 写进源码。
6. 不把凭据提交到 Git。
7. 不把包含凭据的日志上传到公开平台。
8. 一旦怀疑泄露，及时撤销并重新生成。

交易权限、提现权限和只读权限应严格区分。

### 13.4 REST API 是什么？

REST API 通常通过 HTTP 请求与响应完成交互。

例如，获取历史 K 线：

```text
GET /api/market/klines
```

真实的交易所路径、参数和返回格式因平台而异。

典型请求可能包含：

```text
symbol
interval
startTime
endTime
limit
```

响应则包含一系列 K 线数据。

对于低频日线因子研究，REST API 往往已经足够，不必一开始就使用 WebSocket 或付费的高阶数据服务。

### 13.5 WebSocket 与 REST API 的区别

REST 通常是请求—响应模式。

```text
客户端 -> 请求
服务端 -> 响应
```

WebSocket 则可以建立一个持续存在的双向通信通道。

```text
客户端 <==== 持续连接 ====> 服务端
```

它适合：

- 实时行情推送。
- 实时订单状态。
- 实时交易事件。
- 需要持续接收更新的场景。

如果你只需要定时更新日线行情，REST + 定时任务往往更简单。

如果未来需要实时策略、盘口数据或低延迟交易，再考虑 WebSocket 更合适。

### 13.6 什么是 API Rate Limit？

Rate Limit 是 API 调用频率或资源消耗限制。

交易所可能按照请求权重、请求次数、IP、账户或时间窗口限制访问。

如果超出限制，服务端可能返回特定错误码，或者暂时拒绝请求。

因此，数据采集程序应该：

- 理解对应交易所的限流规则。
- 复用客户端连接。
- 设置超时和重试。
- 避免多个任务同时高频请求。
- 合理使用批量接口。
- 记录失败原因。
- 在必要时实施退避策略。

### 13.7 为什么需要 CCXT？

CCXT 是一个用于连接多个加密货币交易所的开源库，提供统一的接口抽象。

例如，某些交易所的原生 API 在参数、返回格式和市场命名方面存在差异。

CCXT 可以帮助你在一定程度上统一这些差异。

示例：

```python
import ccxt

exchange = ccxt.binance({
    "enableRateLimit": True,
})

markets = exchange.load_markets()

ohlcv = exchange.fetch_ohlcv(
    "BTC/USDT",
    timeframe="1d",
    limit=100,
)

print(ohlcv[-1])
```

这只是简化示例。实际市场类型、交易所配置和历史数据范围需要结合具体接口确认。

需要注意：

- `enableRateLimit` 不能代替对所有交易所限制的研究。
- `fetch_ohlcv()` 返回的数据通常需要进行完整性检查。
- 不同市场类型可能具有不同的交易对标识。
- 最后一根 K 线可能尚未收盘。
- 不同交易所的历史数据可用范围可能不同。
- API 成功返回并不代表数据已经符合你的研究口径。

### 13.8 API 返回成功，不代表数据一定正确

假设采集程序下载了某交易对的日线数据。

即使 HTTP 请求成功，也仍然需要检查：

- 时间戳是否正确。
- 时区是否统一。
- K 线是否重复。
- 是否存在缺失日期。
- OHLC 之间是否满足基本关系。
- 成交量是否为合理数值。
- 当前 K 线是否已经完成。
- 交易对是否在目标市场中有效。
- 交易所是否调整了历史数据。

这对于因子研究尤其重要，因为错误的时间戳、重复行或未收盘 K 线都可能造成研究偏差。

---

## 第十四章：Python 项目在服务器上应该怎样组织？

### 14.1 代码、数据、配置和日志必须分清楚

建议把它们视为不同的生命周期。

**代码：** 会随着版本更新而变化。

**数据：** 应长期保存，不能因为代码升级就被覆盖。

**配置：** 需要区分开发、测试和生产环境。

**日志：** 需要轮转、保留和清理。

例如：

```text
/opt/crypto-factor/           # 代码
/srv/crypto-factor-data/      # 行情与因子数据
/etc/crypto-factor/           # 系统级配置
/var/log/crypto-factor/       # 日志
```

不要把所有内容都放在一个目录里，再通过手工操作维护。

### 14.2 环境变量是什么？

环境变量是操作系统进程环境的一部分，常用于传递配置。

例如：

```bash
export APP_ENV=production
```

Python 可以读取：

```python
import os

environment = os.getenv("APP_ENV", "development")
```

常见用途：

- 环境标识。
- API Key。
- 数据目录。
- 日志级别。
- 数据库连接信息。
- 服务端口。

环境变量可以减少硬编码，但不等于自动安全。

如果把秘密写在 Shell 历史、公开日志、进程参数或错误报告里，仍然可能泄露。

对于重要凭据，应该限制配置文件权限，并根据需要使用云厂商的密钥管理服务。

### 14.3 为什么不能把所有配置都写死在代码中？

例如：

```python
DATA_DIR = "/Users/me/Desktop/crypto-data"
```

这个路径在你的本地电脑上可能有效，但迁移到 Linux 服务器后就不再适用。

更合理的方式是把数据目录作为配置：

```python
import os
from pathlib import Path

DATA_DIR = Path(
    os.getenv(
        "CRYPTO_DATA_DIR",
        "/srv/crypto-factor-data",
    )
)
```

这样可以在本地设置一个路径，在服务器上使用另一个路径，而不必修改业务逻辑。

配置的默认值也应符合项目实际情况，不能仅仅因为某个示例路径存在，就认为它一定适合生产环境。

### 14.4 如何管理依赖？

Python 虚拟环境可以隔离项目依赖。

创建：

```bash
python3 -m venv .venv
```

安装依赖：

```bash
.venv/bin/pip install -r requirements.txt
```

对于可重复部署的项目，应尽量锁定依赖版本，或者使用能够生成锁文件的依赖管理工具。

否则，本地和服务器可能安装到不同版本的软件包，导致程序表现不一致。

不要把虚拟环境目录直接从 macOS 复制到 Linux。不同操作系统和 Python 环境之间可能存在二进制兼容问题，通常应在目标环境重新创建虚拟环境并安装依赖。

### 14.5 数据文件为什么要独立管理？

对于你的研究项目：

```text
代码升级
    不应该
      |
      v
导致 Parquet 数据丢失
```

建议：

- 代码通过 Git 管理。
- 数据存放在独立数据目录。
- 重要数据定期备份。
- 数据目录不要随意清空。
- 日志与数据分离。
- 研究结果应具有明确的版本和计算口径。

尤其要注意，DuckDB 数据库文件和 Parquet 文件都可能包含重要的研究结果。不能只备份代码而不备份数据。

---

## 第十五章：构建一个完整的生产部署架构

现在把前面的知识整合起来。

### 15.1 推荐的第一版架构

对于一个小而美的加密货币因子研究项目，可以采用：

```text
                         用户浏览器
                              |
                              v
                    DNS：factor.example.com
                              |
                              v
                    云安全组 / 主机防火墙
                              |
                              v
                         Nginx :443
                              |
                              v
                     FastAPI :127.0.0.1:8000
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
             DuckDB 查询             元数据管理
                  |
                  v
              Parquet 文件

后台定时任务：

cron / systemd timer
          |
          v
Python 数据采集程序
          |
          v
交易所 REST API
          |
          v
原始行情 Parquet
          |
          v
因子计算程序
          |
          v
因子 Parquet
          |
          v
DuckDB 查询与看板
```

这里没有必要一开始就使用 Kubernetes、Kafka、复杂的微服务体系或多台服务器。

对于小规模研究项目，优先把单机部署、日志、数据一致性、自动重启和备份做扎实，通常比提前引入大量基础设施更有价值。

### 15.2 第一步：准备服务器

创建云服务器后：

1. 选择合适的 Linux 镜像。
2. 配置 SSH 公钥或其他安全登录方式。
3. 确认公网 IP 或其他管理入口。
4. 配置安全组。
5. 只开放必要的入站端口。
6. 登录服务器。
7. 更新系统软件包。
8. 创建普通应用用户。
9. 准备项目目录和数据目录。

不要在尚未验证 SSH 登录的情况下贸然关闭现有的登录方式。

### 15.3 第二步：安装 Python 项目环境

安装适合当前系统的 Python 版本。

然后：

```bash
mkdir -p /opt/crypto-factor
cd /opt/crypto-factor

python3 -m venv .venv
```

将项目代码部署到目录中，再安装依赖。

启动之前，先在命令行直接运行一个简单的健康检查，确认解释器、依赖和路径都正常。

### 15.4 第三步：先验证 FastAPI 能够工作

示例：

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def index():
    return {
        "service": "crypto-factor",
        "status": "ok",
    }


@app.get("/api/health")
def health():
    return {"status": "ok"}
```

假设文件位于：

```text
/opt/crypto-factor/app/main.py
```

启动：

```bash
cd /opt/crypto-factor

.venv/bin/uvicorn app.main:app \
    --host 127.0.0.1 \
    --port 8000
```

另开一个 SSH 会话，测试：

```bash
curl -i http://127.0.0.1:8000/api/health
```

如果成功，说明至少在本机回环接口上，服务已经能够接受 HTTP 请求。

这还不能证明公网访问、DNS、HTTPS 或 Nginx 配置已经正常。

### 15.5 第四步：使用 systemd 管理 FastAPI

创建 unit 文件，配置：

- 工作目录。
- Python 解释器。
- 运行用户。
- 监听地址。
- 监听端口。
- 环境变量。
- 重启策略。

启动并检查服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crypto-factor-api
sudo systemctl status crypto-factor-api
```

确认服务可以稳定运行之后，再进入下一步。

### 15.6 第五步：配置域名

在域名的 DNS 管理平台中添加必要记录。

例如：

```text
类型：A
主机记录：factor
记录值：服务器公网 IPv4 地址
```

最终形成：

```text
factor.example.com
```

如果使用 IPv6，还应根据实际网络配置考虑 AAAA 记录。

不要在服务器尚未准备好之前，就认为域名已经能够正确访问。

### 15.7 第六步：安装和配置 Nginx

安装 Nginx 后，配置反向代理。

在证书签发前，可以先检查 DNS 是否指向正确的服务器，以及 HTTP 入口是否能够正常工作。

接着配置 HTTPS 证书。

### 15.8 第七步：启用 HTTPS

通过合适的 ACME 客户端申请证书，并配置 Nginx。

验证：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

然后在浏览器中访问：

```text
https://factor.example.com
```

同时检查证书是否与域名匹配、是否在有效期内，以及 HTTP 是否按预期跳转到 HTTPS。

### 15.9 第八步：部署行情采集与因子计算任务

把任务分为独立的 Python 脚本：

```text
scripts/
├── collect_market_data.py
├── validate_market_data.py
├── build_factors.py
└── check_data_freshness.py
```

任务之间建立明确的依赖关系：

```text
采集行情
   |
   v
验证行情
   |
   v
计算因子
   |
   v
检查结果
```

例如，如果行情采集失败，就不应该无条件继续计算依赖最新数据的因子。

对 Parquet 写入，应尽量避免多个任务同时修改同一个文件。可以通过任务锁、临时文件写入、成功后原子替换，以及清晰的分区管理来减少不一致风险。

### 15.10 第九步：配置定时任务

如果行情每天更新一次，可以使用 cron 或 systemd timer。

例如：

```cron
0 2 * * * /opt/crypto-factor/scripts/run_daily.sh >> /var/log/crypto-factor/daily.log 2>&1
```

在正式依赖该任务之前，应先手工运行脚本，验证：

- 运行用户是否有权限。
- Python 依赖是否齐全。
- 环境变量是否存在。
- 数据目录是否可写。
- 任务失败时是否有明确日志。
- 重复执行是否会造成数据损坏。

### 15.11 第十步：配置监控和备份

最少应关注：

- CPU 使用率。
- 内存使用率。
- 磁盘剩余空间。
- 服务运行状态。
- Nginx 错误日志。
- FastAPI 异常日志。
- 行情数据最新时间戳。
- 因子计算是否成功。
- 证书是否正常续期。
- 备份是否成功。

对重要数据，应该定期执行恢复演练，而不仅仅是确认备份文件存在。

---

## 第十六章：如何排查常见部署故障？

遇到问题时，建议按网络与应用的层次逐层排查，不要一开始就反复修改所有配置。

### 16.1 故障：SSH 无法登录

可能原因：

- IP 地址错误。
- 服务器没有启动。
- 云安全组未放行 SSH 端口。
- SSH 服务没有运行。
- 用户名错误。
- 私钥不匹配。
- 本地网络阻断了连接。

建议顺序：

1. 在云控制台确认实例状态和公网 IP。
2. 确认 SSH 端口和用户名。
3. 检查安全组规则。
4. 检查本地私钥及文件权限。
5. 如果仍然无法登录，使用云厂商提供的控制台或恢复通道排查。

不要为了排查问题，直接把所有管理端口对所有公网地址永久开放。

### 16.2 故障：浏览器访问网站超时

按照以下顺序检查：

```text
域名解析
   |
   v
公网 IP 是否正确
   |
   v
云安全组
   |
   v
主机防火墙
   |
   v
Nginx 是否监听端口
   |
   v
FastAPI 是否运行
```

可使用：

```bash
getent hosts factor.example.com
```

查看系统解析结果。

检查监听端口：

```bash
sudo ss -lntp
```

检查 Nginx：

```bash
sudo systemctl status nginx
```

检查 FastAPI：

```bash
sudo systemctl status crypto-factor-api
```

本机测试：

```bash
curl -i http://127.0.0.1:8000/api/health
```

如果本机请求成功，但公网访问失败，应重点检查 Nginx、监听配置、安全组、防火墙、DNS 和外部网络路径。

### 16.3 故障：502 Bad Gateway

重点检查 Nginx 的上游服务。

```bash
sudo tail -n 100 /var/log/nginx/error.log
```

然后：

```bash
sudo systemctl status crypto-factor-api
sudo journalctl -u crypto-factor-api -n 100
```

最后在服务器内部直接访问 FastAPI。

如果 FastAPI 本身无法工作，优先修复应用，而不是不断修改 Nginx。

### 16.4 故障：HTTPS 证书报错

检查：

- 域名是否与证书匹配。
- 证书是否过期。
- 证书链是否完整。
- Nginx 是否引用了正确的证书路径。
- DNS 是否指向预期服务器。
- 客户端系统时间是否正确。
- 证书续期是否成功。

如果修改了证书路径，应先运行：

```bash
sudo nginx -t
```

再重新加载配置。

### 16.5 故障：程序运行一段时间后停止

检查 systemd：

```bash
sudo systemctl status crypto-factor-api
sudo journalctl -u crypto-factor-api --since "2 hours ago"
```

检查内存：

```bash
free -h
```

检查系统日志和内核日志，判断是否发生了 OOM（内存不足）等事件。不同发行版和权限配置下，相关日志的读取方式可能不同。

同时确认程序是否因未捕获异常、依赖服务不可用或其他原因退出。

不要只设置无限重启。反复重启可能掩盖配置错误，也可能造成资源消耗。

### 16.6 故障：cron 没有按时运行

检查：

```bash
crontab -l
```

并确认：

- cron 服务是否运行。
- 时间与时区是否正确。
- 命令是否使用绝对路径。
- 脚本是否有执行权限。
- 环境变量是否完整。
- 运行用户是否有文件权限。
- 日志中是否有错误。

可以先在与定时任务相同的用户和环境下手动执行脚本。

### 16.7 故障：API 返回 500

500 表示服务端发生了未预期错误，但不能单凭状态码确定根因。

检查 FastAPI 日志，确认异常堆栈。

可能原因：

- 数据文件不存在。
- DuckDB 查询错误。
- Parquet 文件损坏或格式不符合预期。
- 请求参数导致业务逻辑异常。
- 内存不足。
- 外部 API 请求失败。
- 文件权限不足。

生产环境不应将完整的内部异常堆栈、数据库路径、密钥等信息直接返回给普通客户端。

---

## 第十七章：安全基础——部署前必须理解的原则

### 17.1 最小权限原则

程序只应该获得完成工作所必需的权限。

例如：

- FastAPI 不需要 root 权限就能读取 Parquet。
- 行情采集程序不需要修改系统配置。
- 只读行情任务不需要交易所提现权限。
- 数据库不需要向所有公网地址开放。
- 普通网站访问者不应该拥有后台管理权限。

### 17.2 秘密信息管理

需要保护的信息包括：

- SSH 私钥。
- API Key 和 Secret。
- 数据库密码。
- 会话密钥。
- 其他访问凭据。

不要将这些内容直接提交到 Git。

生产环境可以使用受限权限的环境变量文件，或者云厂商提供的秘密管理服务。

例如：

```text
/etc/crypto-factor/api.env
```

应确保只有必要的用户或服务能够读取该文件。

### 17.3 身份认证与授权不是一回事

身份认证（Authentication）回答：

> 你是谁？

授权（Authorization）回答：

> 你有权做什么？

一个用户即使通过身份认证，也不代表他可以执行所有操作。

对于因子看板：

- 普通访问者可以查看公开因子。
- 研究人员可以创建研究任务。
- 管理员可以修改系统配置。

这只是一个示例权限模型。实际设计应根据用户角色和业务风险决定。

### 17.4 HTTPS 不等于 API 安全

即使使用 HTTPS，仍然需要考虑：

- 是否需要登录。
- 是否需要 API Token。
- 是否需要访问控制。
- 是否需要限流。
- 是否允许跨域访问。
- 是否对输入参数进行验证。
- 是否防止敏感信息泄露。
- 是否记录关键操作。

如果因子看板完全公开，只提供只读数据，安全需求可能相对简单。

但如果以后增加了数据删除、任务执行、账户管理或实盘交易功能，安全要求就会明显提高。

---

## 第十八章：把整个系统的工作原理串起来

假设你已经完成了以下部署：

- 一台 Linux 云服务器。
- 一个域名。
- Nginx。
- TLS 证书。
- FastAPI。
- DuckDB。
- Parquet 数据。
- cron 定时采集任务。
- systemd 服务管理。

现在，用户打开：

```text
https://factor.example.com/api/factors?symbol=BTC%2FUSDT
```

完整链路如下。

**第一阶段：域名解析**

浏览器通过 DNS 获取域名对应的地址。

**第二阶段：建立连接**

浏览器通过适当的网络路径连接服务器的 HTTPS 入口。

**第三阶段：TLS 安全握手**

浏览器验证服务端证书并建立加密通信。

**第四阶段：Nginx 处理请求**

Nginx 接收 HTTP 请求，根据域名和路径匹配规则，将请求转发给 FastAPI。

**第五阶段：FastAPI 处理业务**

FastAPI 读取查询参数、执行校验、调用业务代码。

**第六阶段：读取研究数据**

业务代码通过 DuckDB 查询 Parquet 文件，获得需要的数据。

**第七阶段：生成响应**

FastAPI 将查询结果序列化为 JSON，返回给 Nginx。

**第八阶段：返回客户端**

Nginx 将响应通过 HTTPS 连接发送给浏览器。

与此同时，后台定时任务独立运行：

1. cron 到达指定时间。
2. 启动 Python 行情采集脚本。
3. 脚本通过 REST API 请求交易所。
4. 采集程序校验并写入行情数据。
5. 后续因子计算任务生成新的因子数据。
6. FastAPI 在下一次查询时读取更新后的数据。

整个系统由两个主要工作链路组成：

- **在线请求链路：** 浏览器 → Nginx → FastAPI → DuckDB / Parquet。
- **离线数据链路：** 调度器 → Python 采集任务 → 数据校验 → Parquet → 因子计算。

把这两条链路分开理解，是后续扩展项目架构的重要基础。

---

## 第十九章：常用命令速查表

### 19.1 系统与资源

| 目的 | 命令 |
|---|---|
| 查看当前用户 | `whoami` |
| 查看系统版本 | `cat /etc/os-release` |
| 查看运行时间和负载 | `uptime` |
| 查看内存 | `free -h` |
| 查看磁盘空间 | `df -h` |
| 查看目录占用 | `du -sh /path` |
| 查看进程 | `ps aux` |
| 查看网络接口 | `ip addr` |
| 查看路由 | `ip route` |
| 查看监听端口 | `sudo ss -lntp` |

### 19.2 服务管理

| 目的 | 命令 |
|---|---|
| 查看服务状态 | `sudo systemctl status NAME` |
| 启动服务 | `sudo systemctl start NAME` |
| 停止服务 | `sudo systemctl stop NAME` |
| 重启服务 | `sudo systemctl restart NAME` |
| 设置开机启动 | `sudo systemctl enable NAME` |
| 取消开机启动 | `sudo systemctl disable NAME` |
| 重新加载 unit 配置 | `sudo systemctl daemon-reload` |
| 查看服务日志 | `sudo journalctl -u NAME` |
| 持续查看日志 | `sudo journalctl -u NAME -f` |

### 19.3 网络与 HTTP

| 目的 | 命令 |
|---|---|
| 查看 DNS 解析 | `getent hosts DOMAIN` |
| 测试 HTTP 接口 | `curl -i URL` |
| 只查看响应头 | `curl -I URL` |
| 测试 TCP 连接 | `nc -vz HOST PORT` |
| 查看监听端口 | `sudo ss -lntp` |
| 查看 Nginx 配置 | `sudo nginx -t` |
| 重新加载 Nginx | `sudo systemctl reload nginx` |

### 19.4 定时任务与日志

| 目的 | 命令 |
|---|---|
| 编辑当前用户任务 | `crontab -e` |
| 查看当前用户任务 | `crontab -l` |
| 查看最近 100 行日志 | `tail -n 100 FILE` |
| 持续跟踪日志 | `tail -f FILE` |
| 查找关键字 | `grep "ERROR" FILE` |

这些命令应在理解目标和权限的前提下使用。尤其是防火墙、系统服务、文件权限和进程终止相关操作，执行前要确认影响范围。

---

## 第二十章：推荐学习路线与实战检查清单

不建议试图一次性记住所有命令。更有效的方式是按依赖关系逐步学习，并通过一个完整项目进行实践。

### 阶段一：建立网络基础

掌握：

- IP 地址。
- 公网 IP 和私网 IP。
- 端口。
- TCP 和 UDP。
- DNS。
- HTTP 和 HTTPS。
- 本机回环地址。
- 安全组与防火墙。

实践目标：

能够解释为什么 FastAPI 监听 `127.0.0.1:8000` 时，外部用户不能直接访问它。

### 阶段二：掌握 Linux 基础

掌握：

- SSH 登录。
- 文件系统。
- 用户与权限。
- 进程。
- 环境变量。
- 日志。
- 软件安装。
- Python 虚拟环境。

实践目标：

能够独立登录服务器，创建项目目录，部署 Python 依赖并手动启动 FastAPI。

### 阶段三：掌握服务部署

掌握：

- systemd。
- Nginx。
- 反向代理。
- 监听地址。
- 端口转发。
- 服务日志。
- 自动重启。

实践目标：

关闭本地终端后，FastAPI 仍然由 systemd 管理，并且服务器重启后能够按配置启动。

### 阶段四：掌握域名与 HTTPS

掌握：

- DNS 记录。
- A 和 AAAA。
- 域名与 URL。
- TLS。
- 证书链。
- 证书续期。
- HTTP 到 HTTPS 的跳转。

实践目标：

通过自己的域名访问 FastAPI，并且不需要直接暴露应用的 8000 端口。

### 阶段五：掌握自动化任务

掌握：

- cron 或 systemd timer。
- Shell 脚本。
- 退出码。
- 日志重定向。
- 文件锁。
- 重试与幂等。
- 任务失败后的恢复。

实践目标：

让行情采集程序每天自动运行，并能够判断任务是否成功。

### 阶段六：掌握 API 原理

掌握：

- API 契约。
- HTTP 方法。
- 请求头与请求体。
- JSON。
- HTTP 状态码。
- REST。
- FastAPI 路由。
- Uvicorn 与 ASGI。
- 交易所 API 限流。
- API Key 与权限管理。

实践目标：

能够独立设计一个返回因子历史值的 API，并解释从浏览器发出请求到收到 JSON 响应的整个过程。

### 最终部署检查清单

在认为项目已经部署完成之前，逐项确认：

- [ ] 能通过安全的方式登录服务器。
- [ ] 能查看服务器 CPU、内存、磁盘和网络状态。
- [ ] 已建立合理的项目目录和数据目录。
- [ ] Python 依赖可以在目标服务器环境中重建。
- [ ] FastAPI 可以在本机回环地址正常响应。
- [ ] systemd 能正确管理 FastAPI。
- [ ] 服务可以在服务器重启后按预期启动。
- [ ] Nginx 可以将请求转发到 FastAPI。
- [ ] DNS 指向正确的服务器入口。
- [ ] HTTPS 证书有效且与域名匹配。
- [ ] 证书续期机制经过验证。
- [ ] 公网只开放必要的端口。
- [ ] 数据采集任务有明确的调度机制。
- [ ] 定时任务失败时能够留下可定位的日志。
- [ ] 重复运行任务不会破坏已有数据。
- [ ] Parquet、DuckDB 及其他重要数据有备份方案。
- [ ] 已验证数据恢复流程。
- [ ] API 不会意外泄露秘密凭据。
- [ ] 已设置磁盘空间、数据新鲜度和服务状态检查。

---

## 结语：真正需要掌握的是系统之间的关系

学习云服务器运维，不应该止步于记忆 SSH、Nginx、systemd 或 cron 的命令。

你需要形成一套可以迁移到任何项目的思考模型：

**第一，程序在哪里运行？** 由服务器、操作系统和进程管理机制回答。

**第二，别人如何找到它？** 由 IP、DNS、端口和网络路由回答。

**第三，谁可以访问它？** 由监听地址、安全组、防火墙、身份认证和授权机制共同决定。

**第四，访问请求如何被处理？** 由 HTTP、Nginx、ASGI 服务器和应用业务逻辑共同完成。

**第五，通信为什么可信？** 由 TLS、证书验证、身份认证及相关安全机制提供保障。

**第六，程序如何长期稳定运行？** 由 systemd、定时调度、日志、监控、异常处理和恢复机制共同实现。

**第七，数据如何在不同程序之间流动？** 由 API 契约、序列化格式、文件格式、数据库和数据管道共同决定。

当你能够把这些问题连成一条完整链路时，云服务器就不再是一个神秘的黑箱，而是一台可以被你理解、配置、监控和维护的远程 Linux 计算机。

对于你的加密货币因子研究项目，最合适的起点是先完成一台服务器上的完整闭环：**SSH 登录 → Linux 环境准备 → FastAPI 本地监听 → systemd 管理 → Nginx 反向代理 → 域名与 HTTPS → cron 定时采集 → 日志和备份。**

先把这一套基础设施真正跑通，再根据数据量、计算负载和访问规模决定是否增加其他技术组件。这样既能建立扎实的运维能力，也能避免在项目早期引入过多不必要的复杂性。
