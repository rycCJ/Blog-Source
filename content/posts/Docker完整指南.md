---
date: "2025-10-25T09:43:19+08:00"
draft: false
tags: ["", ""]
title: "Docker完整指南" # <--- 修改这一行
summary: "在这里写下您的文章摘要..."
---

本文 Docker 使用场景：电脑 A 上所有环境已经安装好，想要电脑 B 运行电脑 A 中的东西

## Docker 是什么？

### 1. Docker：一个开放源代码的平台

`Docker` 是一个开源平台，它可以让你将应用程序及其所有依赖项（如库、系统工具、代码和运行时）打包到一个标准化的单元中，这个单元被称为容器。 这样，无论在任何环境下，你的应用程序都能以相同的方式运行。

### 2. 容器 (Container)：轻量级的运行环境

容器是一个轻量级、可独立运行的软件包，其中包含了运行某个应用所需的一切：代码、运行时、系统工具、系统库和设置。容器在运行时相互隔离，也与底层的主机系统隔离，确保了环境的一致性。

### 3. 镜像 (Image)：创建容器的模板

镜像是创建 Docker 容器的只读模板。它包含了应用程序代码以及运行该应用程序所需的所有工具、库和依赖项。你可以把镜像看作是容器的“蓝图”或“菜谱”。当你运行一个镜像时，就会创建一个该镜像的实例，也就是一个容器。

### 4. Dockerfile：构建镜像的说明书

Dockerfile 是一个文本文档，其中包含了一系列用户可以执行的指令，用于自动构建 `Docker `镜像。`Docker` 会读取 `Dockerfile` 中的指令来一步步构建镜像。

### 5. Docker 与虚拟机的区别

为了更好地理解 Docker，我们可以将它与传统的虚拟机（VM）进行比较：

| 特性       |                    Docker 容器                    |                       虚拟机 (VM)                        |
| ---------- | :-----------------------------------------------: | :------------------------------------------------------: |
| 虚拟化级别 |         操作系统级别虚拟化，共享主机内核          |      硬件级别虚拟化，每个 VM 都有自己的完整操作系统      |
| 资源消耗   | 资源消耗 轻量级，占用资源少，启动速度快（毫秒级） | 资源密集型，需要加载完整的操作系统，启动速度慢（分钟级） |
| 隔离性     |           进程级别隔离，安全性相对较低            |      完全隔离，每个 VM 都是一个独立的系统，安全性高      |
| 大小       |              镜像通常较小（MB 级别）              |              虚拟机镜像通常很大（GB 级别）               |

简单来说，虚拟机是在模拟一台完整的计算机，而 `Docker` 容器更像是运行在现有操作系统上的一个被隔离的进程。

## 第一步：下载与安装

在两台电脑上下载好 Docker，[视频教程](https://www.bilibili.com/video/BV1THKyzBER6/?spm_id_from=333.337.search-card.all.click&vd_source=de9d0b5f46420ead2e7be875b9558767)
[对应的文本教程}(https://github.com/tech-shrimp/docker_installer) (完成步骤 1，2 即可)

下载完成后，打开终端，在两台电脑测试：

```
docker --version
```

如果能看到版本号，说明一切准备就绪。

## 第二步： 运行第一个 Docker 容器

尝试拉取一个镜像，并且使用镜像运行交互式容器，最后尝试推出并且自动销毁一个容器。

**1.拉取（下载）一个镜像**

镜像是创建容器的模板。我们需要先从一个名为 `Docker Hub` 的公共镜像仓库中获取我们想要的镜像。`Docker Hub` 上存储了成千上万个官方和社区创建的镜像。
现在，拉取官方的 `Ubuntu` 镜像。在终端中输入以下命令：

```
docker pull ubuntu
```

下载完成后，可以使用以下命令查看你本地已经拥有的所有镜像：

```
docker images
```

**2.运行（启动）一个容器**
有了 `ubuntu` 镜像，就可以用它来创建一个容器了。这就像是用 `Ubuntu` 的安装盘（镜像）来启动一台新的电脑（容器）。

在终端中输入以下命令：

```
docker run -it --rm ubuntu bash
```

- docker run: 这是运行一个容器的命令。
- -it: 这是两个参数的组合：
- - -i (--interactive): 保持标准输入（STDIN）打开，允许你与容器进行交互。
- - -t (--tty): 分配一个伪终端或终端。简单来说，就是让你能看到一个命令提示符。
- -it，就是我们常说的“交互式终端”，它能让你像操作一台真实的 Linux 机器一样在容器里输入命令。
- --rm: 这个参数表示当容器退出时，自动将它删除。这对于我们进行快速测试非常方便，可以避免留下很多无用的容器。
- ubuntu: 这是我们想要使用的镜像的名称。
- bash: 这是我们希望在容器启动时执行的命令。bash 是 Ubuntu 系统中最常见的命令行 Shell。

当你执行完这条命令后，你会发现你的命令提示符变了！它可能看起来像这样：

```
root@xxxxxxxxxxxx:/#
```

现在已经在一个运行着 Ubuntu 的 Docker 容器内部了！ 这完全是在 Windows 系统上原生运行的，但你得到的却是一个隔离的、完整的 Ubuntu 文件系统和命令行环境。

**3. 在容器内进行交互**

可以在这个终端里尝试一些常见的 Linux 命令，感受一下：
查看当前目录下的文件： ls
查看操作系统版本： cat /etc/os-release
更新软件包列表： apt-get update

**4.退出容器**
当完成实验后，只需在容器的终端里输入 `exit` 或者按下 `Ctrl + D`，就会退出这个容器。

因为我们之前使用了 --rm 参数，所以在你退出后，这个容器就被自动删除了。你可以通过 `docker ps -a `命令来确认（`-a`会显示所有容器，包括已经停止的），会发现那个容器已经不见了。

## 第三步：定制自己的环境 (Dockerfile)

Dockerfile 是一个文本文件，它像一份“安装说明书”，告诉 Docker 如何一步步地构建一个镜像。我们通过在其中写入指令（如安装软件、复制文件等）来定制我们的环境。

### 核心指令

```
FROM: 指定一个基础镜像，我们的构建将从这个镜像开始。例如 FROM ubuntu:22.04。
RUN: 执行一条命令。主要用于在镜像内部安装软件。例如 RUN apt-get update && apt-get install -y python3。
WORKDIR: 设置工作目录，后续的指令都会在这个目录下执行。例如 WORKDIR /app。
COPY: 将你电脑上的文件或文件夹复制到镜像中。例如 COPY . . 表示将当前目录所有文件复制到镜像的当前工作目录。
CMD: 指定容器启动时默认执行的命令。例如 CMD ["bash"]。
```

### 操作任务

假设你的实验环境需要`Python`和`Git`。现在，我们在电脑 A 上创建一个包含这两个工具的自定义镜像 1.创建一个项目文件夹：在电脑 A 上，创建一个新的文件夹，例如 my-lab-env，然后进入这个文件夹。 2.创建 Dockerfile：在 my-lab-env 文件夹中，创建一个名为 Dockerfile (没有文件后缀) 的文件，用记事本或任何代码编辑器打开它，并输入以下内容：

```Dockerfile
# 使用官方的Ubuntu 22.04作为基础镜像
FROM ubuntu:22.04

# 设置一个环境变量，避免安装时出现交互式提示
ENV DEBIAN_FRONTEND=noninteractive

# 更新软件包列表，并安装python3, pip, 和 git
# 使用 && 连接命令，可以减小镜像层数
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list && \
    apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    git \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# 设置工作目录为 /app
WORKDIR /app

# 当容器启动时，默认执行bash命令，进入交互式终端
CMD ["bash"]
```

3. 构建镜像：打开终端（PowerShell 或 CMD），确保正处于 `my-lab-env `文件夹内。然后执行以下命令来构建第一个自定义镜像：

```Bash 
# -t 参数用来给镜像命名和打标签，格式为 '名称:版本号'
# '.' 表示Dockerfile在当前目录下
docker build -t my-lab-image:1.0 .
```

4. 验证镜像：构建完成后，运行 `docker images`，看到刚刚创建的 `my-lab-image:1.0`。

5. 运行自定义容器：现在，用新镜像启动一个容器：

```Bash 
docker run -it --rm my-lab-image:1.0
```

进入容器后，可以分别输入 `python3 --version` 和 `git --version` 来验证软件是否已经成功安装。然后输入 `exit` 退出。

## 第四步：让数据持久化 (Volumes)

容器本身是“无状态”的，默认情况下，你在容器里做的所有文件修改都会随着容器的删除而消失。为了保存你的工作，我们需要使用**数据卷 (Volume)**。数据卷可以将**主机上的一个文件夹映射到容器里**的一个文件夹。这样，你在容器里对这个文件夹做的任何修改，都会实时同步到电脑的文件夹里，反之亦然。

### 操作任务：

1. 在主机上创建工作区：在电脑 A 的 `my-lab-env `文件夹内，再创建一个名为` workspace` 的子文件夹。这个文件夹将用来存放项目代码。

2. 使用 `-v` 参数运行容器：现在，再次启动容器，但这次加上 -v 参数来挂载数据卷。

```PowerShell
# -v "你电脑上的绝对路径":"容器内的绝对路径"
# Windows CMD:
docker run -it --rm -v "%cd%\workspace:/app" my-lab-image:1.0
#若想把项目换成存放在 D:\my_coding_projects\robotics_simulation
#则上面命令需要修改为：docker run -it --rm -v "D:\my_coding_projects\robotics_simulation:/app" my-lab-image:1.0
# Windows PowerShell:
docker run -it --rm -v "${pwd}\workspace:/app" my-lab-image:1.0
```

- 这个命令的意思是：将当前目录下的 workspace 文件夹，映射到容器内的/app 目录（在 Dockerfile 中设置的工作目录）。

3. 验证数据持久化：

- 进入容器后，你现在正处于 /app 目录下。执行 ls，你会发现它是空的。
- 在容器内创建一个文件：echo "hello from container" > test.txt。
- 现在，不要退出容器，去你电脑的 my-lab-env\workspace 文件夹下查看，你会惊喜地发现里面多了一个 test.txt 文件，内容正是你刚刚写入的！
- 现在，在容器外，用记事本打开这个 test.txt，修改内容为 hello from host 并保存。
- 回到容器的终端，执行 cat test.txt，你会看到内容已经变成了 hello from host。
- 最后，输入 exit 退出并删除容器。但你电脑上 workspace 文件夹里的 test.txt 文件依然存在。你的工作被成功保存了！

## 第五步：分享与同步你的环境 (Docker Hub)

Docker Hub 是一个公共的镜像注册中心，你可以把它想象成 Docker 镜像的 GitHub。你可以将你在本地构建的镜像推送到 Docker Hub，然后在世界上任何一台安装了 Docker 的机器上把它拉取下来。

### 操作任务：

1. 注册 Docker Hub 账号：访问 https://hub.docker.com/ 并注册一个免费账号。记住你的用户名。
2. 在电脑A上登录：打开终端，输入 docker login 命令，然后按照提示输入你的用户名和密码。
3. 给镜像打上正确的标签：要将镜像推送到你的仓库，必须按照 你的用户名/镜像名:版本号 的格式重新给它打标签。

```Bash
# docker tag <本地镜像名:版本号> <你的用户名/远程仓库名:版本号>
docker tag my-lab-image:1.0 your-dockerhub-username/my-lab-env:1.0
```

4. 推送镜像：现在，将你打好标签的镜像推送到 Docker Hub。

```Bash
docker push your-dockerhub-username/my-lab-env:1.0
```

5. 在电脑 B 上拉取镜像：现在，切换到电脑 B。

- 同样，先在终端执行 docker login 登录账号。
- 然后执行`docker pull`命令来拉取你刚刚推送的镜像：

```Bash
docker pull your-dockerhub-username/my-lab-env:1.0
```

## 第六步：网络连接 (Port Mapping)

默认情况下，容器的网络是与外界隔离的。如果在容器里运行了一个需要通过浏览器访问的应用（比如一个 Jupyter Notebook 服务或者一个网站服务器），则需要把容器的端口“暴露”给主机的端口。

### 操作任务：

1. 运行一个 Web 服务：在自定义的容器里运行一个简单的 Python Web 服务器。用电脑 B 上拉取下来的镜像启动容器：

```Bash
# -p <主机端口>:<容器端口>
# 我们将主机的8888端口映射到容器的8000端口
docker run -it --rm -p 8888:8000 your-dockerhub-username/my-lab-env:1.0
```

2. 在容器内启动服务：进入容器后，在 /app 目录下创建一个简单的网页文件：

```Bash
echo "<h1>Hello Docker!</h1>" > index.html
```

然后启动 Python 内置的 HTTP 服务器，它默认在 8000 端口上运行：

```Bash
python3 -m http.server 8000
```

3. 从主机浏览器访问：现在，打开电脑 B 上的任何浏览器（Chrome, Edge 等），访问 http://localhost:8888。 应该能看到 "Hello Docker!" 的字样。
   成功地将容器内的网络服务暴露给了你的主机！

## 第七步：整合所有操作 (最终工作流)

### 在电脑 A 上（当环境有变化时候）

1. 更新环境：如果需要安装新的软件，就去修改 my-lab-env 文件夹里的 Dockerfile
2. 重新构建：修改后，构建一个新的镜像，可以给它一个新的版本号，例如 1.1

```Bash
docker build -t my-lab-image:1.1 .
```

1. **打标签并推送**：给新镜像打上`Docker Hub`标签并推送。

```bash
docker tag my-lab-image:1.1 your-dockerhub-username/my-lab-env:1.1
docker push your-dockerhub-username/my-lab-env:1.1
```

### 在电脑 B 上

1. 拉取最新环境：定期从 Docker Hub 拉取你最新的环境镜像。

```Bash
docker pull your-dockerhub-username/my-lab-env:1.1
```

2. 准备项目文件夹：在电脑 B 上创建一个项目文件夹，例如 `D:\my-projects\project-A`
3. 启动开发容器：使用 docker run 命令，挂载你的项目文件夹，并映射可能需要的端口。

```PowerShell
# 示例: 挂载项目文件夹并映射8888端口
docker run -it --rm -v "D:\my-projects\project-A:/app" -p 8888:8000 your-dockerhub-username/my-lab-env:1.1
```

现在，你可以在这个容器里进行所有的开发、编译、运行等工作。电脑 B 上除了`Docker`之外，不需要安装任何其他的开发工具、库或依赖。代码和文件都安全地保存在 ` D:\my-projects\project-A 文件夹中`
