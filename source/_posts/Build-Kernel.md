---
title: Build a Kernel for Redmi K30 5G by your self.
date: 07-20-2022
---

为自己的 Redmi K30 5G 编译一个 Linux 内核吧~ | 顺便记录一下自己编译内核的历程

<!-- More -->

## 1.系统软件包

不同的系统需要不同的软件包，其实下面的软件包已经足以支撑整个 AOSP 的编译，

1. Ubuntu/Debian 系:

    ```
    sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
    ```

2. REHL/CentOS/Alma/Rocky/Fedora 系:

    ```
    dnf install @development-tools android-tools automake bison bzip2 bzip2-libs ccache curl dpkg-dev gcc gcc-c++ gperf libstdc++.i686 libxml2-devel lz4-libs lzop make maven ncurses-compat-libs openssl-devel pngcrush python python3 python3-mako python-mako python-networkx schedtool squashfs-tools syslinux-devel zip zlib-devel zlib-devel.i686 
    ```

3. openSUSE 系:

    ```
    sudo zypper install bison curl flex git gnupg gperf libesd-devel liblz4-1 ncurses-devel libSDL-devel python-wxWi dgets-devel libxml2-2 libxml2-tools lzop  schedtool squashfs libxslt1 zip zlib-devel make gcc-c++ glibc-devel-32bit ncurses-devel-32bit readline-devel-32bit libz1-32bit && sudo zypper install --type pattern devel_basis
    ```

4. Arch 系:

    如果直接使用 AUR 中提供的 ```aosp-devel``` 包和 ```lineageos-devel``` 包将会非常方便 (下面使用 yay 作为示例)

    注意 您必须启用 ```multilib``` 仓库

    ```
    yay -S lineageos-devel
    sudo pacman -S libxcrypt-compat lib32-libxcrypt
    ```

5. Gentoo/LFS 等用户
    
    我相信你们已经拥有了自己找软件包的能力，嗯

## 2. 交叉编译工具链

众所周知，咱们的电脑是 x86 架构的，如果直接按照 x86 指令集进行编译，那么身为 arm 阵营的手机是理解不了我们编译的内核的，所以我们就需要```交叉编译```这个工具来帮助电脑把他的语言"翻译"成我们的手机可以理解的的语言

我们根据 Google 的行为，选择 ```clang``` 作为我们的交叉编译工具链，这里我们使用我编译的 ```crepuscular clang``` ，或者你也可以使用 ```proton clang``` 等等，大同小异

首先下载 clang :

```
mkdir -p ~/toolchains #或者换成任意你想要把 clang 放在那里的地方

git clone https://gitlab.com/EndCredits/clang-crepuscular.git --depth=1 ~/toolchains/clang-crepuscular #记得要跟上面一样的目录
```

然后为 clang 添加环境变量

将以下内容添加到 ```~/.bashrc``` 或者 ```~/.zshrc```

```
PATH=~/toolchains/clang-crepuscular/bin:$PATH
```

验证一下我们成功了没有，运行结果里有 crepuscular 就好啦

```
clang --version
```

到这里系统的准备工作就结束啦


## 3. 准备内核源码并编译

这里我们使用我修改过的内核源码，从 Github 上 clone

```
git clone https://github.com/EndCredits/android_kernel_xiaomi_sm7250.git -b 12.0-main ~/kernel/sm7250
```

因为我的内核内置了编译脚本，所以可以直接用，其实别的支持打包 dtbo 的 4.19 内核里这个脚本也可以用，只不过需要改一下 defconfig 的名字

```
cd ~/kernel/sm7250
./build.sh all
```

编译完成之后就可以获得一个可以刷入的内核包啦

## 4. 手动编译

其实上面的编译脚本就是把下面这些命令一股脑扔了进去

首先我们进入到内核的源码目录

```
cd ~/kernel/sm7250
```

然后我们会用到 ```make``` 这个工具，首先我们来告诉编译系统需要往我们的内核里编译什么东西，也就是 ```defconfig``` ，如前文所言，这个文件描述了我们的内核里有什么功能，每个机型一般都不一样，你需要找到自己机型对应的配置文件，对于现在的 arm64 机型来说，它一般放置在 ```arch/arm64/configs/``` 下，我们 ```picasso``` 默认的 ```defconfig``` 是 ```arch/arm64/configs/vendor/picasso_user_defconfig``` ，我们略去前面复杂的路径 (因为 ```ARCH=arm64``` 已经告诉了 ```make``` 应该到 ```arm64``` 文件夹中来找这个文件)

```
make ARCH=arm64 LLVM=1 O=../out -j$(nproc --all) vendor/picasso_user_defconfig
```

上面的命令中，```LLVM=1``` 的作用是启用 ```LLVM/clang``` 工具链作为我们的内核编译工具，不然它默认会去寻找 ```gcc``` 作为编译工具，```-j$(nproc --all)``` 是告诉编译系统使用所有 CPU 线程并行编译，```-j```参数后面给出的数字就是你想要同时使用的 CPU 线程数

如果你想要改变编译进内核的东西，你可以使用 ```menuconfig``` 来调整这个文件

```
make ARCH=arm64 LLVM=1 O=../out -j$(nproc --all) menuconfig
```

它会为你打开一个非常漂亮的界面，你可以选择启用或者关闭各项内核的功能，如果没有这方面的需求，直接用默认的就好，然后就可以编译内核啦

```
make ARCH=arm64 LLVM=1 CROSS_COMPILE=aarch64-linux-gnu- CROSS_COMPILE_COMPAT=arm-linux-gnueabi- CLANG_TRIPLE=aarch64-linux-gnu- O=../out -j$(nproc --all)
```

等上一会，内核的编译就完成啦，不过这时候我们得到的内核还只是一个 ```Image```，我们还需要 ```AnyKernel``` 来帮助我们把它刷入手机

```
cd ../out/arch/arm64/boot
git clone https://github.com/EndCredits/AnyKernel3 -b picasso anykernel
```

这里的 ```anykernel``` 是修改过的，只适用于 ```picasso```，如果你想给你的手机也制作一份的话，可以参考这里: 

[ab7be4efcf     anykernel: Adapt to Rosemary kernel for tiffany.](https://github.com/EndCredits/AnyKernel3/commit/ab7be4efcf3b4d11ec33728501d361f63823d393)

[ca166c32e3     adapt to picasso](https://github.com/EndCredits/AnyKernel3/commit/ca166c32e3b98990b4747c446a7252ca6d760460)

然后我们把所有的 dtb 文件都连接成一个，方便我们刷入

```
find ./dts/vendor/qcom -name '*.dtb' -exec cat {} + > ./dtb;
```

然后就可以开始打包啦

```
cp Image ./anykernel/ && cp dtb ./anykernel/ && cp dtbo.img ./anykernel/

zip -q -r Kernel-$(date '+%Z-%Y-%m-%d-%H%M')-$(make kernelversion)-.zip ./anykernel/*
```

这样，内核的打包就到此结束，把上面得到的压缩文件放入手机，就可以愉快的刷入啦

---
