# Docker

### 简介

Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

Docker 从 17.03 版本之后分为 CE（Community Edition: 社区版） 和 EE（Enterprise Edition: 企业版），我们用社区版就可以了。


### 安装

Centos 支持版本7.0及更高版本。

#### 准备

卸载旧版：

> yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate 
> docker-logrotate docker-engine

环境:

> yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2。
>
> yum install -y yum-utils device-mapper-persistent-data lvm2

设置稳定源:

> 阿里云
> 
> yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

> 清华大学
> 
> yum-config-manager --add-repo  https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo

#### 默认

> yum install docker-ce docker-ce-cli containerd.io

默认安装最新版 Dcoker，安装完成后创建好 docker 用户组，但该用户组下没有用户。需要手动启动。

#### 指定版本

1、列出并排序您存储库中可用的版本。此示例按版本号（从高到低）对结果进行排序。

> yum list docker-ce --showduplicates | sort -r
> 
> docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable \
> docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable \
> docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable \
> docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable

2、通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1。

> yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io

#### 验证

启动、关闭

> systemctl start docker \
> systemctl stop docker

通过运行 hello-world 映像来验证是否正确安装了 Docker Engine-Community。

> docker run hello-world


### 使用


































