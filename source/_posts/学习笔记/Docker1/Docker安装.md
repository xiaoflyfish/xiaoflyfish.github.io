---
title: Docker安装
categories: 
- 学习笔记
- Docker1
---

**CentOS 下安装 Docker**

因为 Linux 是 Docker 的原生支持平台，所以推荐你在 Linux 上使用 Docker

要安装 Docker，我们需要 CentOS 7 及以上的发行版本。建议使用overlay2存储驱动程序

如果你已经安装过旧版的 Docker，可以先执行以下命令卸载旧版 Docker

```
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

首次安装 Docker 之前，需要添加 Docker 安装源。添加之后，我们就可以从已经配置好的源，安装和更新 Docker。添加 Docker 安装源的命令如下：

```
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

正常情况下，直接安装最新版本的 Docker 即可，因为最新版本的 Docker 有着更好的稳定性和安全性。你可以使用以下命令安装最新版本的 Docker

```
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

如果你想要安装指定版本的 Docker，可以使用以下命令：

```
$ sudo yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64            18.06.1.ce-3.el7                   docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                   docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos            docker-ce-stable
docker-ce.x86_64            17.09.1.ce-1.el7.centos            docker-ce-stable
```

然后选取想要的版本执行以下命令：

```
$ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

安装完成后，使用以下命令启动 Docker

```
$ sudo systemctl start docker
```

这里有一个国际惯例，安装完成后，我们需要使用以下命令启动一个 hello world 的容器

```
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:7f0a9f93b4aa3022c3a4c147a449bf11e0941a1fd0bf4a8e6c9408b2600777c5
Status: Downloaded newer image for hello-world:latest
Hello from Docker!
```

运行上述命令，Docker 首先会检查本地是否有hello-world这个镜像，如果发现本地没有这个镜像，Docker 就会去 Docker Hub 官方仓库下载此镜像，然后运行它。最后我们看到该镜像输出 "Hello from Docker!" 并退出。

安装完成后默认 docker 命令只能以 root 用户执行，如果想允许普通用户执行 docker 命令，需要执行以下命令 `sudo groupadd docker && sudo gpasswd -a ${USER} docker && sudo systemctl restart docker` ，执行完命令后，退出当前命令行窗口并打开新的窗口即可

**容器技术原理**

Docker 是利用 Linux 的 Namespace 、Cgroups 和联合文件系统三大机制来保证实现的， 所以它的原理是使用 Namespace 做主机名、网络、PID 等资源的隔离，使用 Cgroups 对进程或者进程组做资源（例如：CPU、内存等）的限制，联合文件系统用于镜像构建和容器运行环境

> Namespace

Namespace 是 Linux 内核的一项功能，该功能对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源

Namespace 可以隔离进程 ID、主机名、用户 ID、文件名、网络访问和进程间通信等相关资源。

Docker 主要用到以下五种命名空间。

- pid namespace：用于隔离进程 ID。

- net namespace：隔离网络接口，在虚拟的 net namespace 内用户可以拥有自己独立的 IP、路由、端口等。

- mnt namespace：文件系统挂载点隔离。

- ipc namespace：信号量,消息队列和共享内存的隔离。

- uts namespace：主机名和域名的隔离

> Cgroups

Cgroups 是一种 Linux 内核功能，可以限制和隔离进程的资源使用情况（CPU、内存、磁盘 I/O、网络等）。在容器的实现中，Cgroups 通常用来限制容器的 CPU 和内存等资源的使用。

> 联合文件系统

联合文件系统，又叫 UnionFS，是一种通过创建文件层进程操作的文件系统，因此，联合文件系统非常轻快。

Docker 使用联合文件系统为容器提供构建层，使得容器可以实现写时复制以及镜像的分层构建和存储。

常用的联合文件系统有 AUFS、Overlay 和 Devicemapper 等