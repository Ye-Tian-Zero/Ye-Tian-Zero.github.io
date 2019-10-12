---
title: centos4.3 离线编译 docker
date: 2019-10-11 22:00:58
tags: 
- docker
- container
- compiling
categories:
- build
- docker
top_img: /img/compile_docker_banner.jpg
cover: /img/compile_docker_index.jpg
description: 
- 低版本 centos 如何离线编译 docker
---
> 不要害怕题图，这只是一只可爱的雷兽
# centos4.3 离线编译 docker
由于最近的项目要用 kubernetes，因此产生了需要离线编译 docker 的需求，尤其是要在 centos 4.3 这个古老的发行版上进行编译，这个需求主要是在以下几点需求下催生的：
1. 公司环境老旧，centos 4.3 + 3系内核，且目前无法升级。
2. 内部环境，无法访问外网。
3. docker 如果出问题，可能需要定制性的开发，因此直接使用编译好的 docker 二进制是不可接受的。（我也没试，但我猜官网发布的也没法直接用，毕竟 glibc 库实在是太老了，谁要是这么玩儿过欢迎反馈）

拿到这个问题，第一反应是在 google 上查了查，然而试想一下，docker 最新社区版本是今年发布的，而 centos 4.3 看这里的日期 ([centos4.3](http://vault.centos.org/4.3/)) 看起来大概至少是 10 年前的古老东西。甚至 google 上都没有什么人讨论在这种古老平台上编译 docker 的方式。

所以我记录了这个 blog，用来梳理一遍流程作为备忘，当然如果顺便帮到谁了，那也算积德了。 <span style="color:red">**需要注意的是**</span>，这篇文章只会告诉你在这种环境下编译 docker 的必要条件，而并不会罗列所有的工具在哪下载，怎么安装，了解这些东西最好的方式就是查阅官网，哪怕我写在这里（很多 CSDN 的博客就是这么做的，也产生了这个弊病），你看到的时候可能也已经过期了，或者不是最优的了，最终反而走了歪路。

因此，如果我告诉你前置条件是安装 go，你需要做的是自己去 go 的官网下载二进制包，或者源码编译，或者你喜欢的任何方式都可以，并最终自己确保环境内的 go 是可用的。

**注意：由于原生系统太老了很难直接安装docker，所以官方的 docker 内编译 docker 的手段就像鸡生蛋生鸡一样，难产。因此，这是一篇鲜有的 centos4.3 + docker 外编译 docker 的教程。**

## 前置条件:
- go 环境，依赖的是1.12.5，通常可以用大于等于这个的任意版本
- gcc 8，我用的是 gcc 8，但不是说仅能使用 gcc8，当然具体能用哪些我肯定是没法测试的，只能说越高越好。
- 3系内核，通常 centos 4.3 直接安装后，内核是 2 系的，然而 2 系内核是无论怎样都不可能安装 docker 的，所以你需要自己想办法升级到3系内核，我的版本是 3.10.x
- kernel header，自行上网查找安装 kernel 对应版本的 kernel-header 的方式，通常需要 build kernel 后 make module_install
- [docker](https://github.com/docker/docker-ce) 代码库，我使用的是 docker-ce。包括了 dockerd 以及 docker。
- [containerd](https://github.com/containerd/containerd) 代码库。
- [runc](https://github.com/opencontainers/runc) 代码库。
- 如果你有需求的话（本博客不涉及），[docker-init](https://github.com/krallin/tini) 代码库。
- [libseccomp](https://github.com/seccomp/libseccomp.git)

以上所有的代码库请自行 checkout 到最新版的 tag / branch。比如 docker 目前最新版是 19.03。

# docker 编译
首先，面临的第一个问题是 docker 依赖了 btrfs 和 devicemapper，而系统上没有（实际上大概率你并用不到，甚至这种古老的发行版/内核也不一定支持），解决方案是，裁剪。
## btrfs
btrfs 的裁剪方式是直接删代码，在 docker-ce  根目录删除以下代码：

```bash
rm -rf components/engine/daemon/graphdriver/btrfs   
rm -rf components/engine/daemon/graphdriver/register/register_btrfs.go
```

如果有的话，删除 `components/engine/hack/make.sh` 中与 btrfs 相关的脚本指令，在我这里大概是这段：

``` bash
# test whether "btrfs/version.h" exists and apply btrfs_noversion appropriately
if \
    command -v gcc &> /dev/null \
    && ! gcc -E - -o /dev/null &> /dev/null <<<'#include <btrfs/version.h>' \
; then
    DOCKER_BUILDTAGS+=' btrfs_noversion'
fi
```
你可以在其中直接搜索 btrfs 这个关键词

## devicemapper
devicemapper 相对好处理，你可以像 btrfs 一样手动删除代码，也可以直接用静态的方式编译 docker，这都没有问题。我选择了静态编译的方式，需要做的事情是在 `components/engine/hack/make.sh` 文件中，add_buildtag 这个 function 定义之后，增加一行

```bash
add_buildtag static build
```
或者用你知道的任何方式，在 go 语言 build 的时候增加 static_build 的 tag。我们本次编译依赖使用`components/engine/hack/make.sh` 该文件，因此直接加在其中。

## kernel-header
`components/engine/daemon/graphdriver/quota/projectquota.go` 该文件引用了 kernel-header 中的头文件，不管你之前把 kernel-header 安装到了哪里，请在这个文件最前面通过 cgo 指定路径。

**（你也可以先跳过这一步试试，也许你的系统路径里面就带了需要的定义）**

但总之，在我的系统上，需要做这个改动，即增加下面注释的这一行，让 go 的 c++ 编译器找得到头文件的位置。
```go
package quota // import "github.com/docker/docker/daemon/graphdriver/quota"
/*
#cgo CFLAGS:-I./kernel-header/include // 增加这一行
#include <stdlib.h>
#include <dirent.h>
#include <linux/fs.h>
#include <linux/quota.h>
#include <linux/dqblk_xfs.h>
#ifndef FS_XFLAG_PROJINHERIT
struct fsxattr {
	__u32		fsx_xflags;
	__u32		fsx_extsize;
	__u32		fsx_nextents;
	__u32		fsx_projid;
    unsigned char	fsx_pad[12];
// 以下省略
```
其中，kernel-header 就是我的 kernel-header 的路径，为了方便，我直接把他链接进了这个目录。

## 编译
以上是预处理，以下开始正式编译。
我把编译过程总结了一个脚本，你需要把他放到 docker-ce 代码库的根目录，然后直接执行即可。
### 编译前提
编译之前确保以下信息：
- 正确配置了 GOROOT
- export 的 PATH 中正确包含 go 的 bin 目录和 gcc 工具链

编译脚本：
```bash
#!/bin/sh
set -e
HOME_DIR=$(pwd)
BUILD_PATH="$HOME_DIR/_build/src/github.com/docker/"
mkdir $BUILD_PATH -p
ls $BUILD_PATH | grep docker &>/dev/null || ln -s $HOME_DIR/components/engine $BUILD_PATH/docker
ls $BUILD_PATH | grep cli &>/dev/null || ln -s $HOME_DIR/components/cli $BUILD_PATH/cli
export GOPATH=$HOME_DIR/_build

# compiler dockerd
cd $BUILD_PATH/docker
sh hack/make.sh binary-daemon

# compiler docker
cd $BUILD_PATH/cli
DISABLE_WARN_OUTSIDE_CONTAINER=1 make

cd $HOME_DIR
mkdir -p output/bin
cp $BUILD_PATH/docker/bundles/binary-daemon/dockerd output/bin
cp $BUILD_PATH/cli/build/docker output/bin

rm -rf $BUILD_PATH
```

我保证这个脚本不会删除你的根目录，所以请大胆的

```bash
sh build.sh
# 如有必要，可以 sudo
```

如果顺利的话，你应该会在当前目录的 `output/bin` 下找到 `dockerd` 和 `docker` 这两个二进制，那么恭喜你，docker-ce 编译完成。当然根据讨厌的 `Murphy's Law` 我估计你的环境不太可能如此顺利，因此有问题请留言，只要不是诸如 go 编译器怎么安装或者如何升级 kernel 这样的问题，我看到后会尽量解答（除非我废弃了这个博客）。

# containerd
containerd 是 docker 的运行时，负责 container 的创建和管理，如果没记错的话，应该本身是从 docker 里面分裂出来的项目。所以他是 docker 的依赖之一。如果你上面的 docker 编译成功了的话，我想这个东西的编译就更不太可能出别的幺蛾子了（谁知道呢？）。

回头看一下，[前置条件](#前置条件)，让你安装了 libsecomp 对吧，containerd 需要这个东西，请找到安装他的 include 和 lib 路径在什么地方。

## 指定 libseccomp 路径
就像前面指定 kernel-header 路径一样，你需要在某些文件内指定 libseccomp 的路径。
**（或者你可以试试用 pkg-config 能不能找到 libseccomp 的头文件和 lib ，如果找得到的话，跳过这一步，pkg-config 的机制请自行了解）**

需要修改如下两个文件

```bash
vendor/github.com/seccomp/libseccomp-golang/seccomp.go
vendor/github.com/seccomp/libseccomp-golang/seccomp_internal.go
```

两个文件改法都一样，将这一行
``` go
// #cgo pkg-config: libseccomp
```
修改为
```go
// #cgo CFLAGS: -I./libseccomp/include/ -fPIC
// #cgo LDFLAGS: -L./libseccomp/lib/ -lseccomp -static
```
同样地，
`./libseccomp/include/` 和 `./libseccomp/lib/` 请修改为你自己的 `libseccomp` 的安装目录，我只是为了方便建立了软链到当前目录。

## 修改 MakeFile
如下修改项目根目录下的 `MakeFile`：

将
`GO_BUILDTAGS ?= seccomp apparmor`
修改为
`GO_BUILDTAGS ?= seccomp apparmor no_btrfs static_build`
目的是禁用 btrfs，并静态编译

将
`GO_LDFLAGS=-ldflags '-X $(PKG)/version.Version=$(VERSION) -X $(PKG)/version.Revision=$(REVISION) -X $(PKG)/version.Package=$(PACKAGE) $(EXTRA_LDFLAGS)'`
修改为
`GO_LDFLAGS=-ldflags '-X $(PKG)/version.Version=$(VERSION) -X $(PKG)/version.Revision=$(REVISION) -X $(PKG)/version.Package=$(PACKAGE) -extldflags "-static" $(EXTRA_LDFLAGS)'`
也是为了增加 `-extldflags "-static" ` 这个静态编译参数。

我这里指明了修改内容，如果你看到的 MakeFile 和我的不完全一样，请自行变通。

## 编译
要求和 [编译前提](#编译前提) 一样，请先满足前置需求。
然后建立这个编译脚本：
``` bash
#!/bin/sh
set -e
HOME_DIR=$(pwd)
BUILD_PATH="$HOME_DIR/_build/src/github.com/containerd"
mkdir $BUILD_PATH -p
ls $BUILD_PATH | grep containerd &>/dev/null || ln -s $HOME_DIR $BUILD_PATH/containerd
ls $BUILD_PATH/containerd/vendor/github.com/seccomp/libseccomp-golang | grep libseccomp &>/dev/null || ln -s $HOME_DIR/../libseccomp $BUILD_PATH/containerd/vendor/github.com/seccomp/libseccomp-golang/libseccomp
export GOPATH=$HOME_DIR/_build
cd $BUILD_PATH/containerd
make
mkdir output
cp bin output -r
rm -rf $BUILD_PATH
```

Then

```bash
sh build.sh
# 如有必要，可以 sudo
```

如果顺利的话（又来），你会在 output/bin 里面找到 containerd 和 containerd-shim，和一堆其实不太重要的东西。

# RunC
最后一个 runc，我打赌能活着到这里的人，不会在编译 RunC 上遇到更多问题了，一共就两步。

## libseccomp
没错，runc 也依赖了 libseccomp
请对文件
```bash
vendor/github.com/seccomp/libseccomp-golang/seccomp.go
vendor/github.com/seccomp/libseccomp-golang/seccomp_internal.go
```
执行与 [指定 libseccomp 路径](#%E6%8C%87%E5%AE%9A-libseccomp-%E8%B7%AF%E5%BE%84) 完全一致的修改

## 编译
口干，不多说了，直接上脚本

```
#!/bin/sh
set -e
bcloud local -U
HOME_DIR=$(pwd)
BUILD_PATH="$HOME_DIR/_build/src/github.com/opencontainers/"
mkdir $BUILD_PATH -p
ls $BUILD_PATH | grep runc &>/dev/null || ln -s $HOME_DIR $BUILD_PATH/runc
ls $BUILD_PATH/runc/vendor/github.com/seccomp/libseccomp-golang | grep libseccomp &>/dev/null || ln -s $HOME_DIR/../libseccomp $BUILD_PATH/runc/vendor/github.com/seccomp/libseccomp-golang/libseccomp
export GOPATH=$HOME_DIR/_build
cd $BUILD_PATH/runc
make -f Makefile static
mkdir output/bin -p
cp runc output/bin
rm -rf $BUILD_PATH
```

规矩你们都懂

## Notice
需要注意的是，我之前尝试用 gcc4 编译了 `RunC`，编译成功，但是无法正常运行（错误信息我也记不住了。。。），如果运行docker镜像时遇到 `RunC` 相关问题，请尝试升级编译器。

# dbus & iptables
到这里，所有编译工作已经结束，剩下的问题就是我们古老的系统落后的生产环境了。
为了拉起 dockerd，你最后需要做的一步就是解决 dbus 和 iptables 的问题。

## dbus
dbus 请自行安装升级，官网[dbus](https://www.freedesktop.org/wiki/Software/dbus/)。
这里只有一点需要说的是，如果你 make 出现奇奇怪怪的问题，可能是因为 make 版本过低，请自行升级 make。
还有就是 `configure` 的时候请禁用文档编译，否则你还需要安装一大堆编译文档的工具。（./configure \-\-help 里面写了如何禁用编译文档，我不赘述了）

## iptables
docker 的网桥使用 iptables 的指令比较高级，centos4.3 的版本无法支持。你有两个选择：

1. 也是我的选择，直接禁用 iptables，让 dockerd 使用 hostip，需要在启动 dockerd 的时候增加参数 `--iptables=false`
2. 自行编译安装升级 `iptables`，这个我也踩了一遍雷，结论就是又是一场编译恶战，但是实际上最终是可行的，也能解决问题，只是就不在这里展开了。

所以，请好好评估你是否真的需要这个机制。比如你只有一台机器的话，又或者不需要虚拟的多个 ip 的话，其实完全不需要用这东西。

# 最后
最后，你需要做的就是，把这几个二进放到你的 `PATH` 路径里面:
`dockerd, docker, containerd, container-shim, runc`
然后通过网络上常见的各种 docker 启动脚本配置好你的启动方式，不过 centos 4.3 应该也没有 systemd 可以用，还是老老实实的通过脚本启动 dockerd 试试吧。

最后，请通过 `docker version` 或者 `docker info` 验证功能正确。

# 关于咖啡
最后最后，就是经典的互联网乞讨环节了，如果你觉得我的文章帮助了你，欢迎点击下面的打赏按钮请我喝咖啡，我这边最便宜的小杯热美式一杯10元，上不封顶，下不封底哦:D。
