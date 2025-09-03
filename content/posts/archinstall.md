+++
date = '2025-09-03T09:20:25+08:00'
draft = true
title = 'Arch Linux安装教程（Windows11双系统）'
+++

## 一、叨叨两句

当进入不熟悉的领域，执行不熟悉的工作时，研读目标领域的文档非常有必要，而恰好Arch Linux文档非常齐全，其中[Installation guide](https://wiki.archlinux.org/title/Installation_guide)非常清晰的描述了安装Arch Linux的详细步骤，照本宣科逐步执行即可。读到这里你可能心里咯噔一下大呼上当，这篇文章除了提供一份人人皆知的文档索引，似乎并无任何用处！别急，这篇文章的主角`archinstall`正徐徐向我们走来。

## 二、准备工作

准备工作包括：下载镜像、制作启动盘、设置Windows11

**下载镜像**

[Arch Linux Downloads](https://archlinux.org/download/)页面罗列了各种下载方式和下载资源，选择你最喜欢的下载即可。当然不要忘记校验下载文件的完整性，我用的Windows11，所以是这么校验滴：

```
cd .\Downloads\
get-filehash .\archlinux-2025.08.01-x86_64.iso
```

`get-filehash`会输出一个`SHA256`字符串，同镜像网站提供的`sha256sums.txt`比较一下即可。

**制作启动盘**

[USB flash installation medium](https://wiki.archlinux.org/title/USB_flash_installation_medium)页面罗列了各种操作系统和平台下的制作工具，我选择的是[Rufus](https://rufus.ie)，它提供了一个图形化操作界面，而且不用操心磁盘是否被正确格式化，全程傻瓜式操作，无脑点选即可。

**设置Windows11**

    需要格外注意的是：受限于本人的技术水平和设备状态，本人的选择未必正确，本人的选择未必适合你的设备环境，一切数据丢失和设备损坏的风险和损失需要你个人承担，本人概不负责！如果您愿意冒险请继续阅读，否则请另寻他处！！！

[Dual boot with Windows](https://wiki.archlinux.org/title/Dual_boot_with_Windows)详细描述了Windows11和Arch Linux双系统共存注意事项。由于这篇文档含有不少存在争议的内容，因此哪些设置可以采用，哪些应该避免使用，需要审慎做出选择。

我选择了如下设置：
- 关闭`UEFI Secure Boot`，我的设备上是通过Bios修改
- 关闭`Fast Startup and hibernation`，我的设备上是通过电源管理界面修改的，同时又使用管理员权限执行了`powercfg /H off`命令
- 从磁盘压缩出一块100G的空闲空间，我的设备上是通过Windows自带的磁盘管理操作的
- 我的Windows11的分区现状是：

    | 分区 | 类型 |
    | :------ | :------ |
    |/dev/nvme0n1p1 | EFI System|
    |/dev/nvme0n1p2 | Microsoft reserved|
    |/dev/nvme0n1p3 | Microsoft basic data|
    |/dev/nvme0n1p4 | Windows recovery environment|

    Windows11的4个分区不能动，以免影响Windows11的正常使用。

- 选择`systemd-boot`作为系统启动引导工具，并选择通过`XBOOTLDR`来安装`UEFI Boot Manager`
    - 选择`systemd-boot`是因为它可以自动发现`Windows Boot Manager`并自动添加到启动菜单，非常简单不需要手动维护启动项
    - 选择`XBOOTLDR`是因为Windows的EFI分区容量非常小，无法满足Arch的高频滚动需求，而`XBOOTLDR`则可以通过为Arch添加单独启动分区来规避这一点，你可以根据自己需求设置分区大小，不再受限于Windows启用分区容量，比如我为`XBOOTLDR`分配了1G容量。
    - 关于启动工具配置，可以参考[systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)文档
- 参照[Partitioning](https://wiki.archlinux.org/title/Partitioning)文档的推荐设置，我为Arch Linux规划了3个分区，总的磁盘分区规划为：

    | 分区 | 类型 |
    | :------ | :------ |
    |/dev/nvme0n1p1 | EFI System|
    |/dev/nvme0n1p2 | Microsoft reserved|
    |/dev/nvme0n1p3 | Microsoft basic data|
    |/dev/nvme0n1p4 | Windows recovery environment|
    |/dev/nvme0n1p5 | Linux extended boot|
    |/dev/nvme0n1p6 | Linux swap|
    |/dev/nvme0n1p7 | Linux root (x86-64)|
- 上述分区规划中`/dev/nvme0n1p5`的类型为`Linux extended boot`，这个就是`XBOOTLDR`分区

## 三、安装工作

安装工作包括：启动并设置live环境、配置无线网络、磁盘分区及挂载、执行`archinstall`并自定义Arch Linux

**启动并设置live环境**

在一切准备工作就绪后，我们可以插上启动盘，然后重启系统，等待live环境启动了。这里需要注意要提前调整启动盘的优先级，避免设备直接引导进入Windows系统。

进入live环境之后，映入我们眼帘的是一个terminal，字体非常小，我们先把字体调大以方便查看，命令是`setfont ter-132b`。好，现在看起来就舒服多了，让我们再次检查一下UEFI模式是否正确启用，命令是`cat /sys/firmware/efi/fw_platform_size`，若结果是64则表明系统以UEFI模式启动，且UEFI是64位的，如果不是那么你就要止步于此，回头好好检查一下哪个环节出了问题，让我们就此作别，后会有期！

**配置无线网络**

在上述前置工作完成后，我们先来配置无线网络（不要问为啥一定要用WiFi，问就是没有网线插口），在terminal里配置无线网络，需要借助`iwctl`，具体操作步骤为：
- 执行`iwctl`命令进入配置界面
- 执行`device list`查看无线设备
- 执行`station name scan`扫描WiFi网络，其中name是设备名称，比如wlan0
- 执行`station name get-networks`获取扫描到的WiFi网络SSID列表
- 执行`station name connect SSID`连接指定WiFi网络，在这个过程中需要输入WiFi密码
- 执行exit退出`iwctl`配置界面（不要在这里傻等了，直接exit吧，密码输入正确的话是没有信息提示的，你要问我为啥专门提这个，还能为啥因为我等傻了呗）
- 执行`ping -c 5 archlinux.org`查看网络是否已成功连接（不要问为啥加个-c 5，你要想多等等我也不拦着。啥？你会`ctrl+c`！牛掰，告辞！）

**磁盘分区、格式化及挂载**

首先让我们把分区规划再贴一遍（当然我不是想凑字数，我是要为你节省往前翻再往后翻的时间，就是这么贴心，没办法）：

| 分区 | 类型 |
| :------ | :------ |
|/dev/nvme0n1p1 | EFI System|
|/dev/nvme0n1p2 | Microsoft reserved|
|/dev/nvme0n1p3 | Microsoft basic data|
|/dev/nvme0n1p4 | Windows recovery environment|
|/dev/nvme0n1p5 | Linux extended boot|
|/dev/nvme0n1p6 | Linux swap|
|/dev/nvme0n1p7 | Linux root (x86-64)|

规划是有了，该怎么操作呢，别急，看我装杯！

- 分区：
    - 执行`cfdisk /dev/nvme0n1`进行分区界面（`fdisk`也不是不可以，但我是菜狗就好这口）
    - 选中Free space->选择New->Partition size设置为1G->回车，此时会创建出`/dev/nvme0n1p5`分区
    - 选中`/dev/nvme0n1p5`分区->选择Type->选择`Linux extended boot`
    - 重复以上步骤直到三个分区创建并设置完毕
    - 选择Write写入分区表
- 格式化
    - 执行`mkfs.ext4 /dev/nvme0n1p7`格式化`root`分区
    - 执行`mkswap /nvme0n1p6`格式化`swap`分区
    - 执行`mkfs.fat -F 32 /dev/nvme0n1p5`格式化`XBOOTLDR`分区
- 挂载
    - 执行`mkdir /mnt/boot`，`mkdir /mnt/efi`创建挂载目录
    - 执行`mount /dev/nvme0n1p7 /mnt`挂载`root`分区
    - 执行`mount /dev/nvme0n1p5 /mnt/boot`挂载`XBOOTLDR`分区
    - 执行`mount /dev/nvme0n1p1 /mnt/efi`挂载`Windows EFI`分区
    - 执行`swapon /dev/nvme0n1p6`挂载`swap`分区

**archinstall**

嘚啵嘚说了这么说，总算要迎来本文的主角`archinstall`，然而到这一步反而变简单了，说是傻瓜式操作也不为过，想必这也是`archinstall`出现的意义吧！

在`terminal`直接输入`archinstall`->回车，此时会进入图形化操作界面，按照自己的需求选择相应的配置即可，其中需要注意的是：

- Disk configuration需要设置Mountpoint: /mnt
- Bootloader选择Systemd-boot
- Network configuration选择Use NerworkManager
- 当archinstall执行完毕后，选择chroot into installation for post-installation configurations
- 进入chroot后，执行`bootctl --esp-path=/efi --boot-path=/boot install`完成Systemd-boot配置

## 四、收尾工作

收尾工作包括：还包括啥呀，见仁见智吧，看你想拿Arch Linux干啥了，哈哈哈。。。
