---
title: Pnpm的安装及配置
date: 2025-09-05T15:15:29+08:00
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /images/flower.jpg
images:
  - /images/flower.jpg
  - /images/node-install.png
categories:
  - tutorial
tags:
  - pnpm
  - node
  - XDG
# nolastmod: true
# math: true
draft: false
---

一些关于Pnpm的安装及配置的记录。

<!--more-->

## 扯闲篇

自从第一次接触Node.js环境，我就面临包管理器的艰难选择，本身又是喜欢折腾的性子，免不了要把npm、yarn、pnpm折腾个遍。浪费了不少时间后，才算有所取舍，认定了pnpm，自此，后续所有涉及前端的项目便一直坚定不移的选择pnpm。我在Linux、Windows、WSL2和Mac平台上反复折腾过各种不同的安装方式，结合我个人的设备环境和技术喜好最终选定了一种安装方式持续使用。这种安装方式脱胎于[Node.js官方下载](https://nodejs.org/en/download)页面提供的教程，并根据自己的需要做了些许调整。
![official tutorial](/images/node-install.png)

这种安装方式共分为三个步骤：

1. 通过`nvm`安装`node lts`版本
1. 通过`node`内置的`corepack`启用`pnpm`
1. 通过`pnpm`进行各种必要配置

如果屏幕前的你也认同这种方式，请继续放心食用；如果您不认同，那么后续文字纯粹就是浪费您的宝贵时间，莫让这些扯淡的辞藻污了您的眼睛，请移步他处，去干点啥有意思的事都比阅读这些无病呻吟的文字强得多！

## 通过nvm安装node lts版本

[nvm](https://github.com/nvm-sh/nvm)官方文档提供了很多种安装方式，我选择的是install script，打开terminal，执行命令

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

然后执行命令`nvm install --lts`安装最新的lts版本，执行命令`nvm current`查看当前node版本

## 通过node内置的corepack启用pnpm

Corepack是Node.js官方推出的一个包管理器管理工具，用于管理npm、yarn和pnpm的版本，可以大为简化包管理器安装步骤，基于Corepack你只需要执行一条命令`corepack enable pnpm`即可完成pnpm的下载与安装。

> 注意：从Node.js 25开始，Corepack将不再与未来的Node.js版本一起分发，如果想继续使用Corepack需要自行安装

## 通过pnpm进行各种必要配置

关于pnpm的配置可以参考这些官方文档：

- [environment-variables](https://pnpm.io/pnpm-cli#environment-variables)
- [config](https://pnpm.io/cli/config)
- [globalbindir](https://pnpm.io/settings#globalbindir)

通过corepack安装pnpm虽然简单，很多配置工作还是需要我们自己完成。在Linux或类Unix系统中安装pnpm需要注意XDG环境变量对pnpm配置的影响，这个影响分为两个层面：

- 对pnpm全局配置文件存储位置的影响
- 对pnpm其他“可指定文件存储位置”的配置项的影响，比如`cache-dir`可指定cache存储位置

第一层面影响决定了pnpm全局配置文件存储在哪个位置，pnpm应该去哪里加载全局配置；第二层面影响决定了pnpm可以在全局配置文件里添加的配置项（比如：`cache-dir`、`store-dir`、`state-dir`）的缺省配置值，当你没有通过执行`pnpm config set --global cache-dir /path/to/some/where`命令添加`cache-dir`配置项时，pnpm就通过检查`$XDG_CACHE_HOME`环境变量决定`cache-dir`的默认配置值

首先说第一层面影响，发挥影响的是`$XDG_CONFIG_HOME`环境变量。当我们执行`pnpm config set --global`添加全局配置时，pnpm会做如下工作：

- 首先检查`$XDG_CONFIG_HOME`环境变量
  - 若`$XDG_CONFIG_HOME`环境变量存在，则去`$XDG_CONFIG_HOME/pnpm/rc`这个位置查找配置文件
  - 若`$XDG_CONFIG_HOME`环境变量不存在，则去`~/.config/pnpm/rc`这个fallback位置查找配置文件
- 然后向配置文件中添加全局配置项，比如可以是`global-bin-dir=~/.local/share/pnpm/bin`

然后说第二层面影响，发挥影响的是`$XDG_CACHE_HOME`、`$XDG_STATE_HOME`和`$XDG_DATA_HOME`，下面逐个说明。

对于`$XDG_CACHE_HOME`，它主要影响`cacheDir`配置值：

- 首先检查`$XDG_CACHE_HOME`环境变量
  - 若`$XDG_CACHE_HOME`环境变量存在，则`cacheDir`配置值为`$XDG_CACHE_HOME/pnpm`
  - 若`$XDG_CACHE_HOME`环境变量不存在，则`cacheDir`配置值fallback为`~/.cache/pnpm`

对于`$XDG_STATE_HOME`，它主要影响update checker依赖的`pnpm-state.json`文件位置：

- 首先检查`$XDG_STATE_HOME`环境变量
  - 若`$XDG_STATE_HOME`环境变量存在，则`pnpm-state.json`文件位置为`$XDG_STATE_HOME/pnpm`
  - 若`$XDG_STATE_HOME`环境变量不存在，则`pnpm-state.json`文件位置为`~/.local/state/pnpm`

对于`$XDG_DATA_HOME`，它主要影响`storeDir`、`globalDir`和`globalBinDir`配置值：

- 首先检查`$XDG_DATA_HOME`环境变量
  - 若`$XDG_DATA_HOME`环境变量存在，则
    - `storeDir`配置值为`$XDG_DATA_HOME/pnpm/store`
    - `globalDir`配置值为`$XDG_DATA_HOME/pnpm/global`
    - `globalBinDir`配置值为`$XDG_DATA_HOME/pnpm`
  - 若`$XDG_DATA_HOME`环境变量不存在，则
    - `storeDir`配置值fallback为`~/.local/share/pnpm/store`
    - `globalDir`配置值fallback为`~/.local/share/pnpm/global`
    - `globalBinDir`配置值fallback为`~/.local/share/pnpm`

**需要注意的是：**

- pnpm config生成的global配置文件即便不通过XDG环境变量指定，也可以fallback到默认位置
- `storeDir`、`globalDir`、`globalBinDir`、`stateDir`、`cacheDir`是pnpm全局配置文件里的配置键，可以通过`pnpm config set --global`设置配置值，通过`XDG`环境变量设置配置值，或是使用`XDG`环境变量缺省值
- 在我的WSL2环境中发现，`globalBinDir`必须通过`pnpm config set global-bin-dir ~/.local/share/pnpm -g`手动设置，否则执行`pnpm add $packagename -g`会报错`ERR_PNPM_NO_GLOBAL_BIN_DIR  Unable to find the global bin directory`。当然比起来把整个`~/.local/share/pnpm`路径加入path，我还是更喜欢把`~/.local/share/pnpm/bin`作为`globalBinDir`
