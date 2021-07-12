# 在Docker 中安装Petalinux

# 准备工作

### 安装系统镜像

根据 [Petalinux 2021.1 Package List](https://www.xilinx.com/Attachment/2021.1_PetaLinux_Package_List.xlsx) 可以知道Petalinux支持以下操作系统

* RHEL 7.4, 7.5, 7.6 ,7.7, 7.8, 7.9, 8.1 and 8.2 Server:  Infrastructure Server

* RHEL 7.4, 7.5, 7.6 , 7.7, 7.8, 7.9 , 8.1 and 8.2 Desktop : Development and Creative workstation.
* CentOS 7.4, 7.5, 7.6,7.7, 7.8, 7.9, 8.1 and 8.2 Server (From Everything.ISO): Infrastructure server.
* CentOS 7.4, 7.5, 7.6 ,7.7, 7.8, 7.9, 8.1 and 8.2  Desktop (From Everything.ISO): Development and Creative workstation.
* Ubuntu 16.04.05 and Ubuntu 16.04.06 Desktop.
* Ubuntu 16.04.05  and Ubuntu 16.04.06 Server.
* Ubuntu 18.04.01, 18.04.02, 18.04.03, 18.04.04, 18.04.05, 18.04.06, 20.04 and Ubuntu 20.04.1 Desktop.
* Ubuntu 18.04.01, 18.04.02, 18.04.03, 18.04.04, 18.04.05, 18.04.06, 20.04  and Ubuntu 20.04.1 Server.

可以根据自己的习惯选择一个熟悉的发行版，这里选择了 **centos 8.2** 。查阅 dockerhub 选择了 [centos8.2.2004](https://hub.docker.com/layers/centos/library/centos/centos8.2.2004/images/sha256-cf4f5cf174e78810379036c53fd6d258d13b3735aa5611b0b61e331a8fedbac6?context=explore) 这个镜像。在终端中执行

```bash
$ sudo docker pull centos:centos8.2.2004
```

即可安装镜像。

### 运行docker容器

首先检查一下可用的 docker image

```bash
$ sudo docker image ls
REPOSITORY   TAG              IMAGE ID       CREATED         SIZE
centos       centos8.2.2004   831691599b88   13 months ago   215MB
```

这说明docker里面只有一个image，TAG 是 **centos8.2.2004** 。接下来可以运行基于这个image的container：

```bash
$ sudo docker run --interactive --tty --detach --name petalinux-centos8.2 centos:centos8.2.2004
```

  *  `docker run `的作用是创建一个新的container。
  * `--interactive` 选项的作用是保持标准输入一直处于开启状态。
  * `--tty` 选项的作用是给container分配一个终端。
  * `--detach` 选项的作用是让container在后台运行，并且打印出container的ID。
  * `--name petalinux-centos8.2` 选项的作用是给container起一个名字，如果没有这个选项的，docker会自动给container分配一个名字。


现在检查一下container是否运行成功：

```bash
$ sudo docker container ls
CONTAINER ID   IMAGE                   COMMAND       CREATED         STATUS         PORTS     NAMES
    616bb74dc5b9   centos:centos8.2.2004   "/bin/bash"   6 seconds ago   Up 5 seconds             petalinux-centos8.2
```

这样说明container已经在运行了，下一步要进入container安装一些必要的软件。

### 安装依赖软件包

```bash
$ sudo docker attach petalinux-centos8.2
```

运行这条命令之后就会发现shell已经变成了container的shell了。在进行接下来的操作之前可以参考 [这篇](https://mirrors.tuna.tsinghua.edu.cn/help/centos/) 文章切换一下系统软件源，然后更新一下系统。

```bash
# yum upgrade
```

启用EPEL仓库，

```bash
# yum install epel-release -y
```

安装 DNF

```bash
# yum install 'dnf-command(config-manager)'
```

启用 PowerTools 仓库

```bash
# yum config-manager --set-enabled powertools
```

接下来可以根据[Petalinux 2021.1 Package List](https://www.xilinx.com/Attachment/2021.1_PetaLinux_Package_List.xlsx) 中的 **Quick Installation steps for packages** 部分安装软件包。由于 **pax** 和 **compat-libstdc++-33.i686** 不知道怎么安装，所以暂且先不安装这两个包。安装依赖软件包的命令变为：

```bash
# yum install net-tools gawk make wget tar bzip2 gzip python3 unzip perl patch diffutils diffstat git cpp gcc gcc-c++ glibc-devel texinfo chrpath socat perl-Data-Dumper perl-Text-ParseWords perl-Thread-Queue python3-pip python3-GitPython python3-jinja2 python3-pexpect xz which SDL-devel xterm autoconf libtool.x86_64 zlib-devel automake glib2-devel zlib ncurses-devel openssl-devel dos2unix flex bison glibc.i686 glibc.x86_64 screen glibc-devel.i686 libstdc++.i686 libstdc++.x86_64 rsync
```

此外，还要安装 **tftp** 服务

```bash
# yum install tftp-server
```

还有 passwd 用于修改用户密码

```bash
# yum install passwd
```

和 sudo 用于获取 root 权限

```bash
# yum install sudo
```

### 创建用户

petalinux需要非root用户运行，所以需要创建一个用户用于petalinux程序。使用 adduser 添加新用户

```bash
# adduser xlnx
```

使用 passwd 修改用户密码

```bash
# passwd xlnx
```

使用 usermod 将 xlnx 加入 wheel 组以便于使用 sudo

```bash
# usermod -aG wheel xlnx
```

然后通过 su 命令就可以将用户 切换为xlnx

```bash
# su xlnx
```

## 安装Petalinux

首先切换到用户 **xlnx**，然后创建目录用于存放petalinux

```bash
$ mkdir -p ~/petalinux/2021.1
```

然后将安装文件复制到container中，注意这里的操作是在宿主机中操作

```bash
$ sudo docker cp ~/petalinux-v2021.1-final-installer.run petalinux-centos8.2:/home/xlnx
```

给安装文件加上可执行权限（这里又回到了 container 中操作）

```bash
$ chmod +x petalinux-v2021.1-final-installer.run
```

然后开始安装

```bash
$ ./petalinux-v2021.1-final-installer.run --dir ~/petalinux/2021.1/
```

安装过程中同意三个协议然后等待安装完成即可。

最后可以将 container 变为 image 以便于下次使用。

```bash
$ sudo docker commit -m "petalinux-2021.1 install finished" petalinux-centos8.2 petalinux:2021.1.centos8.2
```

然后可以看到

```bash
$ sudo docker image ls
REPOSITORY   TAG                IMAGE ID       CREATED              SIZE
petalinux    2021.1.centos8.2   73736606c7c5   About a minute ago   15.9GB
centos       centos8.2.2004     831691599b88   13 months ago        215MB
```

petalinux就是根据上面运行的 container 创建的 image。也可以用 export 导出镜像:

```bash
$ sudo docker export petalinux-centos8.2 | gzip > petalinux-2021.1-centos8.2.tar.gz
```

 

