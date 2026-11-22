---
title: "Docker 学习笔记：从 git pull 到镜像化部署"
description: "记录我从 git pull 部署迁移到 Docker 的完整过程：镜像与容器的关系、Namespace/Cgroups/UnionFS 原理、Dockerfile/Volume/网络/Compose，以及本地构建、镜像仓库中转、云服务器上线和远程管理的实操细节"
date: 2026-11-22
lastmod: 2026-11-22
weight: 4
categories:
    - Tutorial
    - Tech Stack
tags:
    - 教程
    - 技术栈
    - docker
    - 部署
---

# Docker 学习笔记：从 git pull 到镜像化部署

- 作者：Haha大王
- 联系方式：2018036661@qq.com
- 创作不易，转载请注明出处，欢迎批评探讨

---

这篇笔记记录我学习 Docker 的完整过程。

起点很具体：之前的项目都是小项目，部署方式就是本地写代码、push 到 GitHub、登录服务器 `git pull`、手动启动。项目变大之后，这套流程开始出现几个绕不过去的问题：

1. **密钥和配置文件不能提交 Git。** 数据库密码、API Key 一旦误提交，风险很高；如果手动在服务器上单独维护一份配置，又很容易版本错乱、丢文件。
2. **环境不一致。** 本地能跑，服务器上的 Python 版本、系统依赖、包版本不一样，就可能报错。项目越大，依赖越多，这个问题越明显。
3. **部署步骤是“人肉脚本”。** 换一台新服务器，要重新装运行时、装依赖、建目录、配环境变量，步骤全靠脑子记。
4. **升级和回滚麻烦。** 出问题想快速切回上一个稳定版本，`git` 回滚再重新启动并不轻松。

Docker 正好对应这四个痛点。下面按我实际学习的顺序整理：先建立宏观认知，再拆原理，然后落到这套博客项目之外的一个 Python + MySQL 部署例子上，最后把免费额度、远程管理和常见坑补齐。

> 说明：文中涉及 Docker 套餐、Docker Hub 额度和价格的部分，是我学习时的官方规则；这些数字会调整，实际使用前以 Docker 官网为准。

## 一、先建立宏观认知：Docker 到底是什么

### 1.1 虚拟机与容器不是一回事

新手最容易混淆的是虚拟机和 Docker 容器。

**虚拟机（VMware、KVM 等）**是在物理机里虚拟出一整台完整的电脑：有独立内核、完整操作系统，体积通常几个 G，启动要几十秒到几分钟，资源开销很大。

可以类比成**租一套完整的毛坯房**：水电、墙面、地板全都单独给你一套。

**Docker 容器**不是虚拟机。它不虚拟出完整操作系统，而是在宿主机已有的 Linux 内核之上做一层隔离，把程序本身和它需要的依赖打包在一起。

可以类比成**一个带全部家具的独立集装箱**：码头（宿主机）只有一套地基和水电（宿主机内核），但每个集装箱内部物品独立、互不干扰。集装箱很小，开箱即用，几秒就能启动。

两者的关键区别是：**容器共享宿主机内核**。容器里不能换内核，也不能运行与宿主机内核不兼容的东西；虚拟机则有自己独立的内核。

### 1.2 镜像与容器：模板和实例

Docker 里两个最基础的对象必须分清：

- **镜像（Image）**：只读的“安装包 / 模板”，由一层层文件系统叠加而成，本身不能运行。
- **容器（Container）**：镜像运行起来的实例，是一个被 Linux 内核隔离起来的进程组。一个镜像可以启动很多个容器。

可以用蛋糕来类比：镜像是配方加做好的蛋糕胚；容器是把蛋糕胚拿出来、抹上奶油开始售卖的那个具体蛋糕。

也可以用 DVD 来类比：镜像是只读的光盘，容器是播放器。同一张光盘可以放进很多台播放器同时播放；播放时产生的临时数据（音量、字幕设置）不会写回光盘。

这里有一个非常重要的推论：**容器是临时的**。容器启动时会在只读镜像层之上加一层可读写层，程序产生的日志、临时文件默认只写在这一层；容器删除后，这一层也随之丢失。数据库文件、上传文件、重要业务数据不能只放在容器内部，必须通过数据卷挂载到宿主机。

### 1.3 一句话定义

**Docker 是一个打包、分发、运行程序的工具：把代码、依赖和运行环境打包成镜像，再用镜像启动隔离的容器。同一个镜像，在任何装了 Docker 的机器上，运行环境一致。**

### 1.4 密钥应该放在哪里

这正好对应我最初最关心的问题：**密钥、敏感配置不要放进镜像。**

镜像只放代码和依赖；数据库密码、API Key、`.env` 配置文件通过环境变量或宿主机挂载注入。这样 GitHub 仓库是干净的，镜像里也没有密钥。

但这里有一个必须提前纠正的误区：**只在 `docker-compose.yml` 里写挂载，并不能阻止 `.env` 被打进镜像。** 如果 Dockerfile 里写了 `COPY . .`，而项目根目录又存在 `.env`，它照样会进入构建上下文、被复制进镜像。正确做法是同时使用 `.dockerignore` 把 `.env` 排除掉，后文会给出完整配置。

## 二、Docker 的工作原理

### 2.1 三大内核能力

Docker 的隔离能力主要依托 Linux 内核的三项机制。

**1. Namespace：隔离**

Namespace 给容器单独隔离进程、网络、文件系统、用户 ID、主机名等视图。容器里看到的是一个“独立小系统”：容器 A 看不到容器 B 的进程和文件，容器内部也看不到宿主机的全部进程。

但要注意：**容器共享宿主机内核**。Namespace 做的是视图隔离，不是硬件虚拟化。

**2. Cgroups：限制资源**

Cgroups 控制一个容器最多能用多少 CPU、内存、磁盘 I/O，防止某个服务把整台服务器吃光。例如可以给一个量化服务容器限制最多 2 核 CPU、4 GB 内存。

**3. UnionFS：分层文件系统**

这是 Docker 镜像体积小、可复用的核心。镜像不是一个大文件，而是一层层只读层叠加起来的：

- 第一层：基础层，例如 `python:3.10-slim`，底层是精简的 Linux。
- 第二层：安装依赖，例如 `pip install pandas numpy`。
- 第三层：复制项目代码。
- 第四层：写入启动命令等元数据。

每一层都可以缓存和复用。如果两个项目都基于 `python:3.10-slim`，这一层在服务器上只存一份，不会重复下载。

容器启动时，Docker 会在所有只读层之上再加一层**可读写层**。程序写日志、写临时文件都落在这一层；容器删除后默认丢失。

这就是为什么数据库数据必须用 Volume 持久化。

### 2.2 一次 docker run 发生了什么

把三项能力串起来，`docker run` 大致做了这些事：

1. Docker Daemon 找到本地镜像；没有的话先去 Registry 拉取。
2. 基于镜像的只读层，创建一层新的可读写层。
3. 调用 Linux Namespace 创建隔离环境。
4. 用 Cgroups 限制 CPU、内存等资源。
5. 在隔离环境里执行镜像中定义的入口命令（`CMD` 或 `ENTRYPOINT`）。
6. 这个正在运行的隔离实例就是容器。

### 2.3 Docker 的 C/S 架构

Docker 是客户端 / 服务端结构：

- **Docker Client（客户端）**：我敲命令的地方，例如 `docker run`、`docker build`、`docker pull`。
- **Docker Daemon（`dockerd`，服务端）**：后台常驻进程，真正负责管理镜像、容器、网络和存储。
- **Docker Registry（镜像仓库）**：存放镜像的地方。官方公共仓库是 Docker Hub，也可以自建私有仓库（例如 Harbor）。

客户端把命令发给 Daemon，Daemon 干活。

### 2.4 Docker 与 Git 的区别

这两个东西经常被类比，但职责不同：

| 维度 | Git / GitHub | Docker / Registry |
| --- | --- | --- |
| 存什么 | 源代码，文本，人可读 | 镜像，分层文件包，已包含运行环境 |
| 拿到之后 | 还要装依赖、配置环境、启动 | 直接 `docker run` 启动容器 |
| 版本能力 | 分支、commit、merge、历史 | 主要是 tag，没有 Git 的分支和合并语义 |
| 是否必需 | 源码管理必需 | Registry 是可选的，可以本地构建、离线传输 |
| 典型用途 | 协作开发、代码托管 | 构建产物分发、环境交付 |

**Docker Hub 不是必须用的。** 可以本地 `docker build` 之后用 `docker save` 导出 tar 包，`scp` 到服务器再 `docker load`；也可以自建私有 Registry，镜像完全不经过外网。

### 2.5 Mac / Windows 上的 Docker Desktop

Mac 和 Windows 本身不是原生 Linux，所以 Docker Desktop 会在系统里跑一个轻量 Linux 虚拟机，在虚拟机里运行 `dockerd`。日常使用的 `docker` 命令和 Linux 服务器完全一样，基本无感。

Linux 服务器上则不需要 Docker Desktop，只装 Docker Engine（纯命令行）。

一句话记：

- 自己的 Mac / Windows：装 **Docker Desktop**，带图形界面。
- 远程 Linux 服务器：装 **Docker Engine**，只有命令行。

## 三、Docker 之后的部署流程

先看改造后的流程，就能理解它为什么能解决密钥问题。

旧流程：

> 本地写代码 → push GitHub → 服务器 `git pull` → 手动启动程序

新流程：

1. 本地开发，把密钥、配置文件（`.env`）加入 `.gitignore`，永远不提交 GitHub。
2. 本地编写 `Dockerfile`，只描述怎么复制代码、安装依赖，**不写密钥**。
3. 本地执行 `docker build`，构建镜像。
4. 可选：把镜像推送到镜像仓库（Docker Hub 私有仓库或自建仓库）。
5. 到服务器拉取最新镜像，**不再 `git pull` 源码**。
6. 启动容器时从外部注入密钥和配置：
   - 方式 A：环境变量，例如 `--env KEY=xxx`；
   - 方式 B：把宿主机上单独存放的 `.env` 或配置文件挂载进容器。
7. 新版本上线：拉新镜像，停旧容器，用新镜像启动；回滚时重新启动旧版本镜像。

这套流程的好处：

- GitHub 仓库干净，没有任何密钥。
- 本地、测试机、服务器使用同一个镜像，环境一致。
- 容器销毁不会带走密钥文件，因为密钥在宿主机单独目录。
- 项目依赖再多，打包一次就可以到处运行。

GitHub 仍然可以保留，用来管理源代码；只是**部署的交付物从源码变成了镜像**。

两种部署方式对比如下：

| 方案 | 交付物 | 敏感密钥 | 环境一致性 | 回滚 |
| --- | --- | --- | --- | --- |
| `git pull` 部署 | 源代码 | 容易误提交到仓库 | 差，依赖要在服务器现场安装 | `git` 回滚后重启 |
| Docker 部署 | 镜像 | 通过挂载或环境变量外部注入，不进镜像 | 高，同一个镜像在哪运行都一致 | 启动旧版本镜像 |

## 四、四个核心概念

### 4.1 Dockerfile：构建镜像的剧本

`Dockerfile` 是放在项目根目录的纯文本文件，描述如何构建镜像。

一个极简的 Python 项目示例：

```dockerfile
# 基础镜像，复用官方做好的 Python 环境（第一层）
FROM python:3.10-slim

# 设置容器内工作目录
WORKDIR /app

# 先复制依赖清单，利用分层缓存：依赖不变，这一层就不用重新构建
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制项目代码进去
COPY . .

# 容器启动时执行命令
CMD ["python", "main.py"]
```

构建命令：

```bash
docker build -t myproject:v1 .
```

`-t myproject:v1` 给镜像打标签：名字 + 版本号。版本号很重要，它相当于部署产物的版本标识，方便升级和回滚。

但上面的 Dockerfile 有一个隐患：`COPY . .` 会把项目根目录的所有东西都复制进去，包括 `.env`、`.git`、本地数据文件。必须配合 `.dockerignore`：

```text
# .dockerignore
.env
.env.*
.git
.gitignore
__pycache__/
*.pyc
node_modules/
*.tar
.DS_Store
```

`.dockerignore` 不只是“减小构建上下文”，它首先是一道安全边界：**不把密钥送进构建上下文，就不会有机会被打进镜像。**

另外，不要把密钥写成 `ARG` 或 `ENV` 再 `RUN` 使用。即使后面删掉，密钥仍可能留在某一层的历史里；镜像层是叠加的，删除文件不等于删除历史。

### 4.2 Volume：解决容器数据易失

容器内部的可读写层是临时的。Volume 用来把宿主机的目录或 Docker 管理的卷挂载到容器里：

```bash
docker run -v /宿主机/路径:/容器内路径 myproject:v1
```

典型用法：

- 数据库数据目录挂载 Volume，数据库容器删了，数据文件还在宿主机。
- 把宿主机上的 `.env` 挂载到容器 `/app/.env`，容器程序读取这个文件，密钥始终留在服务器本地，不会打进镜像。

挂载配置文件时建议加只读标志，避免容器里的程序误改宿主机配置：

```bash
docker run -v /宿主机/.env:/app/.env:ro myproject:v1
```

### 4.3 网络

Docker 自带网桥网络。默认情况下，同一个 Compose 网络里的容器可以互相用**服务名**通信。

例如有一个 MySQL 容器和一个后端容器，后端直接访问 `mysql:3306`，不需要知道宿主机 IP。

端口映射则把宿主机端口转发到容器内部端口：

```bash
docker run -p 8000:8000 myproject:v1
```

这里要区分“容器之间通信”和“对外暴露”：

- 容器之间通过服务名和内部端口通信，不需要 `ports`。
- 只有需要从宿主机或公网访问的服务，才映射端口。
- **数据库端口不要直接暴露到公网。** 后端的 MySQL 只需要在 Compose 内部网络里可达；把 3306 映射到 `0.0.0.0` 并放行安全组，等于把数据库暴露给整个互联网。

### 4.4 Docker Compose

当项目不止一个程序，例如后端服务 + MySQL + Redis，逐个敲很长的 `docker run` 很麻烦。Docker Compose 用一个 `docker-compose.yml` 描述多个容器、挂载、环境变量、网络和依赖关系。

一键启动：

```bash
docker compose up -d
```

一键停止：

```bash
docker compose down
```

注意现在的命令是 `docker compose`（中间是空格），属于 Docker CLI 的 Compose v2 插件；老教程里的 `docker-compose`（带短横线）是独立 Python 工具，新环境一般不需要单独安装。

一个最小示例：

```yaml
services:
  app:
    build: .
    image: myproject:v1
    ports:
      - "8000:8000"
    volumes:
      # 挂载宿主机本地的 .env 配置（密钥在这里，不上 Git）
      - ./.env:/app/.env:ro
    restart: always
```

`.env` 里写数据库密码、API Key，并加入 `.gitignore`。

对个人项目、中小型服务来说，Compose 已经够用，暂时不需要上来就碰 Kubernetes。K8s 解决的是大规模集群编排，学习成本和使用成本都高得多。

### 4.5 Docker 不解决什么

为了避免把 Docker 当成万能工具，有几条边界要提前记住：

1. **Docker 不是安全沙箱。** 容器是进程隔离，不是强安全隔离；不要指望它防止入侵。容器共享宿主机内核，逃逸风险和内核漏洞直接相关。
2. **Docker 不负责写代码，也不负责业务逻辑。** 它解决的是环境交付、部署和配置解耦。
3. **容器里尽量只跑一个主进程。** 不要把 SSH、定时任务、后台脚本全塞进同一个容器；一个容器一个职责，日志和生命周期才好管理。
4. **不要在容器里手动改代码调试。** 镜像应该是不可变的；改代码要重新构建新镜像，否则下次重建容器时改动就消失了，也无法复现。

## 五、新手最容易卡住的六个问题

### 5.1 要不要下载？Mac 能用吗

Mac 完全能用，官方工具叫 **Docker Desktop**，需要下载安装。

官网：<https://www.docker.com/>

这里有两个东西容易混：

1. **Docker Desktop**：本地客户端套装，包含 Docker Engine、图形界面 Dashboard、`docker` 命令行和 Compose。Mac（Intel / Apple Silicon）和 Windows 都用它。
2. **Docker Engine**：纯引擎，没有图形界面，专门给 Linux 服务器用。

所以：

- 自己的 Mac 笔记本：装 Docker Desktop。
- 远程 Linux 服务器：装 Docker Engine，不装 Desktop。

Mac 上的 Docker Desktop 会在后台跑一个轻量 Linux 虚拟机，`dockerd` 跑在虚拟机里；终端里的 `docker` 命令和这个虚拟机里的 Docker 服务通信。对使用者来说基本无感。

### 5.2 有图形界面吗

两者都有。

Docker Desktop 自带 Dashboard，可以可视化查看镜像、运行中的容器、日志、端口，也能启动、停止、删除容器，适合新手先建立直观感受。

但生产环境和服务器部署的行业标准仍然是命令行 + Compose YAML。图形界面是辅助工具，核心能力都在命令行。

如果服务器上也想要网页版图形界面，可以装 Portainer（本身也是一个 Docker 容器），后文会讲。

### 5.3 Docker Registry 和 GitHub 像吗

大方向很像，但有一个关键区别。

- GitHub 存放**源代码（文本）**，拉下来之后还要安装依赖、编译、启动，人可以直接阅读源码。
- Docker Hub / Registry 存放**镜像（分层文件包）**，镜像里已经装好依赖、代码和运行环境，拉下来直接启动成容器。

相同点：

- 都分公共仓库和私有仓库。
- 都用 tag 标记版本，例如 `:v1`、`:v2`，类似 Git 的 tag。

不同点：

1. GitHub 是源码版本管理，有分支、commit、merge；Registry 只是镜像存储和分发服务，没有这些 Git 能力。
2. 不是必须使用 Docker Hub。可以本地构建后用 `docker save` 导出、`scp` 传输、`docker load` 加载；也可以自建 Harbor 等私有 Registry。
3. Docker 软件本身不依赖 Docker Hub 才能运行。Docker Hub 只是可选的镜像托管网站。

镜像里通常包含源码，但它不是给人阅读和修改的形态；`docker save`、`docker history` 等命令仍然可以查看镜像内容，所以**镜像不是加密的**，不能把密钥当成“藏在镜像里就安全”。

### 5.4 镜像为什么不能直接“安装”

这是当时最大的卡点。

Windows 的 `.exe` 或 macOS 的 `.dmg` 安装包，作用是**把文件复制到系统、注册服务**，安装后文件留在主机系统里，成为系统的一部分。

Docker 镜像不是这种安装包。**镜像是一个只读的、静态的文件集合**，它本身不能运行，只是一堆文件加上启动指令。

`docker run` 的本质是：

1. 拿只读镜像；
2. 在上面叠加一层可读写层；
3. 调用 Namespace / Cgroups 创建隔离环境；
4. 在隔离环境里执行镜像写好的启动命令，启动进程。

这个正在运行的隔离实例才是容器。

为什么设计成这样：

- 一个镜像可以同时启动多个独立容器，例如同一个后端镜像开 3 个实例做负载均衡。
- 容器是临时隔离沙盒，销毁时只删掉上层读写层，镜像保留，随时可以再建一个干净实例。
- 镜像不可变，不会因为运行被改坏。

如果“直接把镜像安装到宿主机”，镜像里的文件和依赖会污染服务器系统，又回到了环境地狱，Docker 最大的意义就没了。

### 5.5 Docker 公司提供服务器才能运行吗

Docker, Inc. 是一家商业公司，开发维护 Docker Desktop、Docker Engine，也运营 Docker Hub。但**Docker 引擎本身运行不需要连接 Docker 公司的服务器**。

Docker Engine 是本地或服务器上独立运行的程序，离线也可以 `build` 镜像、`run` 容器。

只有执行 `docker pull python:3.10` 这类命令时，才会去 Registry 拉镜像。完全可以离线使用，或者换成私有仓库。

Docker 也不是 pip / npm 那种包管理器：

- pip 的核心职责是下载软件包、安装到本机环境，会修改本机系统文件。
- Docker 的核心职责是基于镜像创建隔离进程环境。
- `docker pull` 看起来像 `pip install`，但它只是下载镜像文件，**不会把软件安装进宿主机系统**，软件运行在隔离容器里。

### 5.6 Mac 和远程服务器怎么配合

核心结论：**远程服务器必须安装 Docker Engine。** SSH 只是登录服务器终端的手段，和 Docker 本身没有关系。

两种工作模式：

**模式 A：分开构建（推荐）**

1. Mac 本地写代码、写 Dockerfile，本地 `docker build` 并测试。
2. 把镜像推到镜像仓库（Docker Hub 私有仓库 / 自建 Registry）。
3. SSH 登录服务器。
4. 在服务器执行 `docker pull`，再 `docker run` 或 `docker compose up` 启动。

Mac 上的 Docker 和服务器上的 Docker 是两个独立的 Docker 服务，互不通信，通过镜像仓库传递镜像。

**模式 B：远程 Docker Daemon（不推荐新手）**

Docker Client 可以直接连接远程服务器上的 `dockerd`，在 Mac 终端敲命令，直接指挥服务器构建、启动容器。

这不是默认的 SSH 隧道，而是让 `dockerd` 开放 TCP 端口（通常是 2375 无加密、2376 TLS）。**公网直接开放 2375 非常危险，等于把服务器 root 权限交出去。** 如果一定要用，应该走 SSH 隧道，后文会讲。

新手阶段建议老老实实用模式 A：本地 build 测试 → push 仓库 → 服务器 pull 启动。

如果不想经过任何镜像仓库，也可以本地导出 tar 包：

```bash
docker save myproject:v1 -o app.tar
scp app.tar 用户名@服务器IP:/home/用户名/my_project/
```

服务器上加载：

```bash
docker load -i app.tar
```

适合不想上传外网、私有化要求高的小项目；缺点是版本管理麻烦，每次升级都要重新传包。

### 5.7 把整条链路串起来

```text
MacBook（本地）
├─ Docker Desktop：docker 客户端 + dockerd 引擎 + GUI 面板
└─ 写 Dockerfile → docker build 生成镜像 → 本地 docker run 测试
        ↓（可选 push 上传镜像）
Docker Hub / 私有 Registry（镜像存储，可选）
        ↓（服务器 docker pull 拉取）
远程 Linux 服务器
├─ 安装 Docker Engine（dockerd，无 GUI）
└─ 拿到镜像 → docker run 创建隔离容器（进程）对外提供服务
```

SSH 只负责让我连上服务器终端，敲服务器上的 `docker` 命令。

## 六、完整实战：把本地项目部署到云服务器

下面用一个具体场景走一遍完整流程。

### 6.1 前置约定

现状：

- Mac 本地已经跑通一个项目。
- 项目包含 Python 后端、Node 前端、依赖库、`.env` 密钥配置，以及 MySQL 数据库。
- 本地 Mac 和云服务器 Ubuntu 上都还没装 Docker。
- 目标是用 Docker 上线，密钥不进镜像、不进 Git。

方案选型：`Dockerfile + docker-compose`。

两条路线：

- **路线 A（推荐，标准流程）**：本地构建镜像 → 私有 Docker Hub 中转 → 服务器拉镜像启动。
- **路线 B（不走外网仓库）**：本地 `docker save` 导出 tar → `scp` 到服务器 → `docker load` 加载。

两条路线都遵守同一条规则：**`.env`、密钥文件永远不打进镜像、不提交 Git；镜像只打包代码和依赖，配置和密钥单独挂载。**

数据库有两种做法：

- 方案 A：数据库也跑在 Docker 容器里，用 Compose 一起管理，新手容易上手。
- 方案 B：数据库使用云厂商的 RDS，正式上线更省心，不用自己维护数据库容器。

下面演示方案 A：Python 后端 + MySQL 容器。

### 6.2 项目结构

先把项目整理成类似这样：

```text
your_project/
├── .gitignore          # 必须写
├── .dockerignore       # 必须写，防止 .env 进入镜像
├── .env                # 密钥配置，不上 Git、不打进镜像
├── Dockerfile          # 打包后端 Python 服务
├── docker-compose.yml  # 编排 app + mysql
├── requirements.txt    # Python 依赖清单
├── package.json        # Node 依赖（如果前端需要）
├── main.py             # Python 入口
└── src/                # 源码目录
```

`.gitignore` 至少包含：

```text
.env
__pycache__/
node_modules/
*.tar
.DS_Store
```

`.dockerignore` 至少包含：

```text
.env
.env.*
.git
.gitignore
__pycache__/
*.pyc
node_modules/
*.tar
.DS_Store
```

它们保护的是不同环节：`.gitignore` 防止密钥进 Git，`.dockerignore` 防止密钥进镜像构建上下文。

### 6.3 本地：安装 Docker Desktop

1. 打开 <https://www.docker.com/>，下载对应 Apple Silicon 或 Intel 的 Mac 版本。
2. 双击 `.dmg` 安装，拖进应用程序，打开 Docker Desktop。
3. 首次启动按提示授权，等待顶部状态栏出现鲸鱼图标。
4. 新开终端验证：

```bash
docker --version
docker compose version
docker info
```

`docker compose` 是内置命令，不需要再单独安装旧版 `docker-compose`。

### 6.4 本地：导出依赖清单

进入项目根目录：

```bash
pip freeze > requirements.txt
```

如果前端有独立的 Node 项目，保留已有的 `package.json` 和 lock 文件即可。

注意架构差异：如果 Mac 是 arm64，而云服务器是 amd64，构建镜像时需要指定目标平台，后文会讲。

### 6.5 本地：编写 Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 先复制依赖，利用分层缓存
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制源码；.dockerignore 已经排除 .env
COPY . .

CMD ["python", "main.py"]
```

如果项目同时有 Node 前端，建议拆成两个容器、两个 Dockerfile：前端容器负责构建静态资源，后端容器跑 API。这里先聚焦 Python 后端。

### 6.6 本地：编写 docker-compose.yml

```yaml
services:
  app:
    build: .
    image: my-project:v1
    ports:
      - "8000:8000"
    volumes:
      # 把宿主机的 .env 只读挂载进容器，密钥从这里注入
      - ./.env:/app/.env:ro
    restart: always
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    volumes:
      # 数据库数据卷，容器删了数据还在
      - mysql_data:/var/lib/mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

几个细节：

- 新版 Compose 会忽略 `version:` 字段，所以这里不写。
- MySQL 没有写 `ports`。后端通过 Compose 内部网络的 `mysql:3306` 访问它，不需要把 3306 暴露到宿主机或公网。本地调试如果确实要用客户端连，可以临时写成 `127.0.0.1:3306:3306`，只绑定本机。
- `depends_on` 只控制启动顺序；加了 `condition: service_healthy` 之后，Compose 会等 MySQL 健康检查通过再启动 app。
- `MYSQL_ROOT_PASSWORD` 和 `MYSQL_DATABASE` 从同目录的 `.env` 读取，Compose 会自动加载 `.env` 做变量替换。
- 如果应用代码直接读环境变量，而不是自己读取 `.env` 文件，可以把 `volumes` 换成 `env_file: .env`；按代码实际读取配置的方式选择，两种方式不要同时混用而不自知。

### 6.7 本地：编写 .env

```env
# .env 示例
DB_HOST=mysql
DB_PORT=3306
DB_NAME=mydb
DB_PASSWORD=SuperStrongPass123
API_SECRET=xxxx你的密钥xxxx
```

这个文件只存在本地和服务器，不进 Git，也不进镜像。

### 6.8 本地：联调测试

在项目根目录执行：

```bash
docker compose up -d
```

- `-d` 表示后台运行。
- 打开 Docker Desktop 面板，可以看到 app 和 mysql 两个容器。
- 访问 `127.0.0.1:8000` 验证项目。
- 查看日志：

```bash
docker compose logs -f app
```

测试完成后停止：

```bash
docker compose down
```

此时 `my-project:v1` 镜像已经构建在本地 Docker 里。

需要清空数据库卷时要用 `docker compose down -v`，但这会删除数据，执行前必须确认。

### 6.9 分支 A：推送到私有仓库，服务器拉取

#### A1. 准备 Docker Hub

1. 打开 <https://hub.docker.com/> 注册账号。
2. 创建一个 Private Repository，名字假设为 `myrepo`。
3. 本地登录：

```bash
docker login
```

输入用户名和密码。更推荐在账号设置里创建 Access Token，用 Token 代替密码登录；在共享机器上使用后执行 `docker logout`。

#### A2. 给镜像打标签

刚构建的镜像名是 `my-project:v1`，推送前要改成 `用户名/仓库名:版本`：

```bash
docker tag my-project:v1 你的hub用户名/myrepo:v1
```

#### A3. 推送镜像

```bash
docker push 你的hub用户名/myrepo:v1
```

推送完成后，Docker Hub 私有仓库里只有镜像，`.env` 和密钥没有上传。

#### A4. 服务器安装 Docker Engine

SSH 登录服务器：

```bash
ssh 服务器用户名@服务器公网IP
```

使用 Docker 官方脚本安装：

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

执行 `usermod` 后必须**断开 SSH 重连一次**，否则当前用户没有 Docker 权限，每次都要 `sudo`。

另外要意识到：加入 `docker` 组基本等同于拥有 root 级权限，因为可以直接挂载宿主机目录、启动特权容器。服务器账号本身要做好密钥登录和最小权限管理，不要把 `docker` 组随便加给不受信任的用户。

重连后验证：

```bash
docker --version
docker compose version
```

#### A5. 服务器上准备目录和配置

```bash
mkdir ~/my_project && cd ~/my_project
```

服务器上**不需要源码，也不需要 `git pull`**，只需要两个文件：

1. `docker-compose.yml`
2. `.env`

先在服务器创建 compose：

```bash
nano docker-compose.yml
```

粘贴内容，但要做两处修改：

- 删除 `build: .`。服务器上没有源码，留着它会让 Compose 尝试在服务器构建镜像。
- 把 `image` 改成 `你的hub用户名/myrepo:v1`。

服务器版大概是这样：

```yaml
services:
  app:
    image: 你的hub用户名/myrepo:v1
    ports:
      - "8000:8000"
    volumes:
      - ./.env:/app/.env:ro
    restart: always
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    volumes:
      - mysql_data:/var/lib/mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

然后在服务器手动创建 `.env`：

```bash
nano .env
```

填入与本地一致的密钥配置。这个文件不要通过不安全的渠道随便传，手动输入或使用受控的密钥管理方式更稳妥。

#### A6. 拉取并启动

```bash
docker login
docker pull 你的hub用户名/myrepo:v1
docker compose up -d
```

查看日志：

```bash
docker compose logs -f app
```

访问 `服务器公网IP:8000`，项目上线。

云服务器安全组只需要放行 8000（如果前面有 Nginx，则放行 80/443）；**不要放行 3306**。数据库只在 Compose 内部网络可达。

后续更新流程：

1. 本地改代码，重新 `docker build`，打新 tag（例如 `v2`）。
2. `docker push` 新镜像。
3. 服务器修改 compose 里的镜像 tag，`docker compose pull`，再 `docker compose up -d`。
4. 回滚时把 tag 改回旧版本，重新 `pull` + `up -d`。

不要长期只用一个 `latest` tag。版本号或 git commit sha 才能让回滚有明确目标。

### 6.10 分支 B：tar 包传输，不经过镜像仓库

适合不想把镜像上传到外网的项目。

本地导出：

```bash
docker save my-project:v1 -o my-project-v1.tar
```

传输到服务器：

```bash
scp my-project-v1.tar 服务器用户名@服务器IP:/home/你的用户名/my_project/
```

服务器加载：

```bash
cd ~/my_project
docker load -i my-project-v1.tar
```

然后同样准备只含 `image` 的 compose 和 `.env`，执行 `docker compose up -d`。

全程镜像不经过第三方仓库。缺点是版本管理麻烦，每次升级都要重新导出、传输、加载。

### 6.11 常见坑

**坑 1：Mac arm64 构建的镜像在 amd64 服务器上报 `exec format error`**

M 系列 Mac 默认构建 arm64 镜像，云服务器通常是 amd64。构建时指定目标平台：

```bash
docker build --platform linux/amd64 -t my-project:v1 .
```

推送和运行都使用这个镜像。Docker Desktop 自带 buildx，支持跨平台构建，但跨平台构建可能比本地构建慢。

**坑 2：把 `.env` 打进镜像**

只在 Compose 里写挂载是不够的。Dockerfile 的 `COPY . .` 会复制构建上下文里的所有文件；如果 `.env` 在项目根目录，它就可能进入镜像。必须同时配置 `.dockerignore`，并且绝对不要写 `COPY .env .`。

**坑 3：在 Dockerfile 里用 `ARG` / `ENV` 传密钥**

构建参数和镜像历史可能被保留下来。密钥应该在运行时通过环境变量或挂载注入，不要进入构建阶段。

**坑 4：把数据库端口暴露到公网**

MySQL 只应该在后端容器所在的内部网络里可达。删除 `ports: 3306:3306`，安全组也不要放行 3306。需要本地连接时，只绑定 `127.0.0.1`，并通过 SSH 隧道访问。

**坑 5：以为 `depends_on` 会等 MySQL 准备好**

普通 `depends_on` 只保证启动顺序，不保证服务已经可以接受连接。使用 `condition: service_healthy` 配合 `healthcheck`，或者在应用里实现连接重试。

**坑 6：服务器 compose 里保留了 `build: .`**

服务器上没有源码，Compose 会尝试构建并失败，或者误用旧缓存。服务器版 compose 只保留 `image`。

**坑 7：容器删了才发现数据在容器里**

数据库、上传文件必须挂载 Volume。执行 `docker compose down -v` 前要确认数据已经备份。备份可以在宿主机执行 `docker compose exec mysql sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" "$MYSQL_DATABASE"' > backup.sql`，恢复时把 SQL 导回容器；只备份 Volume 目录也可以，但必须保证数据库处于一致状态。

**坑 8：安全组只放了应用端口，却忘了数据库、监控、Portainer 的边界**

只放行真正需要公网访问的端口。管理面板（Portainer 9000）和 Docker Daemon（2375/2376）不应该直接暴露在公网。

## 七、免费与付费：哪些功能需要花钱

先划一条最重要的分界线：

- **Docker Engine（Linux 服务器上的纯引擎）本身开源免费**，没有功能阉割，也不受订阅限制。
- 收费的主要是 **Docker Desktop（Mac / Windows 桌面客户端）、Docker Hub 的额度、Docker Scout 安全扫描、Build Cloud 云构建、官方支持和团队权限**等附加服务。

免费套餐叫 **Docker Personal**，付费套餐通常分为 **Pro / Team / Business**。对个人学习、个人项目和小型量化项目来说，免费版大多数场景够用。

再次提醒：下面是学习时的套餐内容和价格，Docker 会调整规则，实际以官网为准。

### 7.1 Docker Personal 免费包含什么

**1. Docker Engine（Linux 服务器）**

- `docker run / build / pull / push / volume / network` 全部核心能力。
- `docker compose` 完整功能。
- 容器、镜像、数据卷、网桥网络、自定义网络都没有额外限制。
- 可选开启内置 Kubernetes。

服务器上部署项目，这部分始终免费。

**2. Docker Desktop（Mac / Windows）**

免费许可通常覆盖：

- 个人自用学习、个人项目。
- 教育、学生、教师。
- 非商用开源项目。
- 员工少于 250 人且年收入低于 1000 万美元的小公司 / 初创公司。

超过这个规模的企业商用 Docker Desktop 才需要购买订阅。

免费版 Desktop 的 GUI、镜像管理、容器日志、端口预览都正常；Compose、本地构建、本地运行容器也没有功能阉割。

**3. Docker Hub 额度**

学习时的免费额度大致是：

- 无限个公开镜像仓库。
- 1 个私有镜像仓库，存储上限约 2 GB。
- 登录账号后每小时 100 次 pull；未登录 IP 每小时 10 次。
- 可以打 tag、使用有限的自动构建额度。

对只有一个私有项目镜像的场景，1 个私有仓库刚好够用。

**4. Docker Scout**

- 1 个仓库支持持续漏洞扫描，可以看依赖包的 CVE 和 SBOM 清单。

**5. 社区资源**

官方文档、社区论坛、社区支持；没有工单 SLA。

### 7.2 免费版的限制

1. Hub 私有仓库只有 1 个。
2. Hub pull 有速率上限，高并发 CI 自动部署时容易被限流。
3. Scout 只能持续扫描 1 个仓库。
4. Build Cloud、Testcontainers Cloud 只有试用时长。
5. 没有审计日志、细粒度 RBAC、SSO、企业级安全管控。
6. 没有官方付费工单支持。

### 7.3 付费三档的主要差异

学习时的参考价格：

- Pro：约 9 美元 / 人 / 月。
- Team：约 15 美元 / 人 / 月。
- Business：约 24 美元 / 人 / 月。

**Pro（单人专业版）**

- 无限私有镜像仓库，单个私有仓库上限约 5 GB。
- Docker Hub 取消 pull 限流。
- Scout 2 个仓库持续扫描。
- Build Cloud 每月 200 分钟、Testcontainers Cloud 100 分钟。
- 5 个工作日响应的官方支持。
- 更高级的文件同步和 Desktop 调试能力。

**Team（小团队）**

Pro 全部内容，外加：

- 最多 100 个成员、组织级 RBAC（谁能 push、谁只能 pull）。
- 审计日志。
- Scout 不限仓库扫描。
- 每月 500 分钟 Build Cloud + Testcontainers。
- 2 个工作日 SLA。

**Business（大企业）**

Team 全部内容，外加：

- SSO、SCIM 账号同步。
- 镜像访问策略、企业镜像白名单、加固版 Desktop、增强容器隔离。
- 不限组织数量、每月 1500 分钟云构建。
- 1 个工作日紧急支持、发票结算、合规报告。

### 7.4 对我的实际结论

1. 服务器上的 Docker Engine 永远免费，可以放心用。
2. Mac 上的 Docker Desktop，个人学习、个人项目免费直接用。
3. Docker Hub 免费版 1 个私有仓库，刚好够当前项目；公开仓库不限。
4. 如果以后有第二个独立项目也需要私有镜像仓库，不一定要买 Pro，可以自建 Harbor，或使用其他容器镜像服务。
5. 免费版最大的实际痛点是 Hub pull 限流；如果写 GitHub Actions 频繁拉镜像，高峰期可能触发限速，自建仓库可以绕开。
6. 付费功能主要是团队协作、企业安全、高并发 CI 和更多私有仓库，现阶段不需要付费。

还要纠正两个常见误解：

- “Docker 收费了，容器就会跑得更快”是错的。容器运行、镜像构建的性能和订阅无关。
- 可以完全不用 Docker Hub：本地 `save` 成 tar，传到服务器 `load`，全程不碰 Hub，自然不受 Hub 免费额度限制。

## 八、在本地管理远程 Docker

部署上线之后，自然会产生一个需求：能不能在 Mac 的 Docker 面板里管理服务器上正在运行的项目？

结论：**默认不行。** Mac 本地的 Docker Desktop 默认只管理本机 Docker，看不到云服务器上的容器。

有两种方案可以实现远程管理。

### 8.1 方案一：本地客户端连接远程 dockerd

`docker` 命令是 C/S 架构，客户端可以不连本机 `dockerd`，而是连远程服务器上的 `dockerd`。连上之后，本地所有 `docker` 命令操作的都是服务器上的容器和镜像。

远程 `dockerd` 默认只监听本机 Unix socket，不对外开 TCP。要远程连接，需要让它监听：

- `tcp://0.0.0.0:2375`：无加密，**强烈不推荐**。
- `tcp://0.0.0.0:2376`：TLS 加密，配置复杂。

公网直接开放 2375 非常危险：任何人知道地址就能操控服务器 Docker，基本等同于拿到 root 权限。

更安全的做法是 **SSH 隧道转发 Docker socket**：

```bash
ssh -L localhost:23750:/var/run/docker.sock 你的服务器用户名@服务器IP
```

然后在本机指定 Docker 主机：

```bash
export DOCKER_HOST=tcp://localhost:23750
```

此时：

- 终端里的 `docker ps` 看到的是服务器上的容器。
- Docker Desktop 也可以切换到这个远端上下文，用图形面板管理远程容器。

缺点是：

1. 隧道断开就失效，每次使用都要重新开。
2. 适合个人临时调试，不适合长期稳定管理。
3. 本地执行构建时，构建动作实际发生在远程服务器上，不是在本机构建；这与“本地 build → push 仓库 → 服务器 pull”的流程不同。

Docker 里的 **Context** 就是用来保存多个 Docker 服务地址、快速切换本机和远程的。也可以直接使用 SSH 传输：

```bash
docker context create server --docker "host=ssh://你的服务器用户名@服务器IP"
docker context use server
docker ps
```

Docker CLI 会通过 SSH 连接远程 `dockerd`，不需要手动维持端口转发。切回本机：

```bash
docker context use default
```

### 8.2 方案二：在服务器上装 Portainer

这是更推荐给个人项目的方案，避开了开放 2375 的风险。

Portainer 本身就是一个 Docker 容器，运行在服务器上，提供网页版管理界面。

服务器上执行：

```bash
docker run -d \
  -p 127.0.0.1:9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  --restart always \
  portainer/portainer-ce
```

说明：

- `portainer-ce` 是社区版，永久免费，个人使用足够。
- 这里把端口绑定在 `127.0.0.1`，不直接暴露到公网。需要访问时用 SSH 隧道：

```bash
ssh -L 9000:localhost:9000 你的服务器用户名@服务器IP
```

然后浏览器打开 `http://localhost:9000`。

- 如果确实要从公网访问，至少要在安全组里限制来源 IP，并启用 HTTPS，不要长期裸奔。
- 第一次访问设置管理员账号密码。

功能：

- 查看所有容器、启停、日志、进入容器终端。
- 查看镜像、Volume、网络。
- 管理这台服务器上的 Compose 服务。
- 可以添加多台服务器，在同一个网页里管理多台机器的 Docker。

优点：

1. 不需要开放 Docker 的 2375 危险端口。
2. 任何电脑、手机浏览器都能访问。
3. 不依赖 SSH 隧道是否一直在线。
4. 个人开发者和小项目使用很普遍。

缺点：

- 它是独立网页，不是 Mac 自带的 Docker Desktop 面板。
- 仍然要保护好管理员账号和 9000 端口。

### 8.3 两种方案对比

| 方案 | 能否使用 Mac 原生 Docker 面板 | 安全程度 | 适合场景 |
| --- | --- | --- | --- |
| SSH 隧道 + Docker Context | 可以，原生 Dashboard 可切换 | 较高，流量走 SSH 加密 | 临时调试、熟悉 Docker Context |
| Portainer 网页 | 不能，使用独立网页 | 较高，推荐绑定本机或限制来源 IP | 长期管理服务器项目、看日志、启停容器 |

### 8.4 几个提醒

1. 不要为了方便直接给 `dockerd` 开公网 2375。网上很多老教程这么写，公网裸奔等于送服务器。
2. 区分两件事：
   - 镜像发布流程：本地 build → push Registry → 服务器 pull 启动。这个流程不需要远程管理 Docker。
   - 远程图形管理：上线之后查看、启停、看日志的手段。它和镜像发布是两套东西。
3. 即使使用 Portainer，也不建议在网页里直接改代码、构建镜像。最佳实践仍然是本地构建、镜像分发。
4. 管理面板的端口、Docker socket 的访问权限，都是高权限入口，安全等级要和 SSH 登录同级对待。

## 九、学习路线与检查清单

### 9.1 建议的上手顺序

1. 在 Mac 上安装 Docker Desktop，熟悉基础命令：

```bash
docker images        # 看本地镜像
docker ps            # 看正在运行的容器
docker ps -a         # 看所有容器（包括已停止的）
docker run           # 启动容器
docker build         # 构建镜像
docker logs 容器id    # 看程序日志
docker rm / rmi      # 删除容器 / 删除镜像
```

2. 跑一个 hello world，确认整条链路：

```bash
docker run hello-world
```

这条命令会去 Docker Hub 拉取 `hello-world` 镜像，启动容器，输出文字后退出。可以直观看到 pull（下载只读包）和 run（创建容器运行）的区别；镜像会留在本地。

3. 写一个极简 Python 脚本 + Dockerfile，本地 build，本地跑容器。
4. 学习 Volume 挂载，把 `.env` 挂载进去，练习**密钥和镜像分离**。这是本次学习的首要目标。
5. 学习 Docker Compose，管理多服务。
6. 熟悉服务器部署流程：构建镜像、推送镜像、服务器拉取、启动。
7. 进阶：私有镜像仓库、镜像瘦身、GitHub Actions 自动构建。

CI 是下一步的升级：本地 push 代码到 GitHub，GitHub Actions 自动 build 镜像、推送仓库，服务器拉新镜像，全自动完成，不再手动 `git pull` 或手动 build。

### 9.2 上线前检查清单

- [ ] `.env` 在 `.gitignore` 和 `.dockerignore` 里都有。
- [ ] 镜像历史里没有密码、Token、私钥。
- [ ] 数据库数据挂载了 Volume，并验证过备份和恢复。
- [ ] 数据库端口没有暴露到公网。
- [ ] 服务器只保留 `image`，没有 `build: .`。
- [ ] `depends_on` 配合了 `healthcheck`，应用有连接重试。
- [ ] 镜像使用明确版本 tag，不是长期 `latest`。
- [ ] 安全组只放行必要的端口。
- [ ] Docker socket 和管理面板（Portainer 9000）没有裸奔在公网。
- [ ] 回滚流程演练过：改回旧 tag，`pull` + `up -d`。

## 结语

学完这一圈，我对 Docker 的理解可以压缩成四句话：

1. **镜像是只读模板，容器是它的运行实例。** 容器删了，写在可读写层的数据就没了。
2. **容器共享宿主机内核。** 它轻量、启动快，但不是虚拟机，也不是强安全沙箱。
3. **配置和密钥在运行时注入。** 镜像只放代码和依赖；`.env` 既不进 Git，也不进构建上下文。
4. **部署的交付物从源码变成镜像。** GitHub 继续管源码，Registry 或 tar 包负责交付镜像，服务器只负责拉取和运行。

这四句话对应的，就是我从 `git pull` 迁移到 Docker 的全部动机：环境一致、密钥分离、升级可回滚、服务器不再需要现场装依赖。
