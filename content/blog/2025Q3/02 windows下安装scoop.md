---
title: windows下安装scoop
date: 2025-10-22
authors:  
  - name: lunuj
    link: https://github.com/lunuj
tags:
  - 2025@3
  - 开发环境
---
在 **Windows** 下，搭建开发环境是一个比较复杂的问题，而 **Scoop** 可以统一开发环境，并安装常见的开发软件。
<!--more-->

## 安装 Scoop
[Scoop](https://scoop.sh/) 需要 [Windows PowerShell 5.1](https://aka.ms/wmf5download) 或者 [PowerShell](https://aka.ms/powershell) 作为运行环境，如果你使用的是 Windows 10 及以上版本，Windows PowerShell 是内置在系统中的。而 Windows 7 内置的 Windows PowerShell 版本过于陈旧，需要手动安装新版本的 PowerShell。

查看 **powershell** 版本：
```
$PSVersionTable.PSVersion
```

安装 **Scoop**：
```powershell
# 设置 PowerShell 执行策略
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
# 下载安装脚本 可从 https://get.scoop.sh 手动下载
irm get.scoop.sh -outfile 'install.ps1'
# 执行安装, --ScoopDir 参数指定 Scoop 安装路径 
.\install.ps1 -ScoopDir 'C:\Scoop'
```

**Scoop** 的默认镜像地址是[GitHub](https://github.com/ScoopInstaller/Scoop)，可以更改默认镜像源：
```powershell
# 更换scoop的repo地址 
scoop config SCOOP_REPO https://mirror.nju.edu.cn/git/scoop-main.git
# 拉取新库地址 
scoop update
```

## Scoop 分支
**Bucket** 在 **Scoop** 中类似于一个软件仓库或软件包集合，用于管理和分发软件安装包及其配置信息。Bucket包含了软件包的元数据，如软件名称、版本、下载链接、校验码等，**Scoop** 通过这些信息来下载、安装和管理软件。

**Scoop** 默认提供了一个主 **Bucket**（Main Bucket），其中包含了大量常用的命令行程序和工具。除了默认 **Bucket** 外，**Scoop** 还支持丰富的社区共享 **Bucket**，如Extras Bucket，其中包含了更多种类的软件，包括图形界面程序等。

默认是 `master` 分支，想要切换到其他分支，可执行如下命令：
```powershell
# 切换分支到develop
scoop config scoop_branch develop
# 重新拉取
scoop update
```

**Bucket** 操作
```powershell
# 查看当前bucket
scoop bucket list
# 添加 extras bucket
scoop bucket add extras https://mirror.nju.edu.cn/git/scoop-extras.git
# 删除已安装的bucket
scoop bucket rm extras
```

## Scoop 使用
Scoop 常用命令：
```powershell
# 安装软件 
scoop install appname
# 卸载软件：
scoop uninstall appname
# 卸载软件并删除相关数据：
scoop uninstall appname -p
# 搜索软件：
scoop search appname
# 安装scoop-search插件后通过插件搜索：
scoop install main/scoop-search
scoop-search appname
# 查看已安装软件：
scoop list
# 更新Scoop：
scoop update
# 更新单个软件：
scoop update app
# 更新所有软件：
scoop update -a
# 禁止更新：
scoop hold appname
# 解除禁止更新：
scoop unhold appname
# 清理旧版本软件：
scoop cleanup appname
# 查看所有安装文件：
scoop cache
# 删除指定软件安装文件：
scoop cache rm appname
# 删除所有安装文件：
scoop cache rm -a
```
## 常用软件安装
```powershell
# 安装Chrome 126（支持MV2）
scoop install extras/googlechrome@126.0.6478.183

# 安装Git
scoop install main/git

# 安装HiBit Uninstaller
scoop install extras/hibit-uninstaller

# 安装Koodo Reader
scoop install extras/koodo-reader

# 安装LocalSend
scoop install extras/localsend

# 安装Sublime Text 4169
scoop install extras/sublime-text@4-4169

# 安装Microsoft VS Code
scoop install extras/vscode

# 安装Magpie
scoop install extras/magpie

# 安装Caesium Image Compressor
scoop install extras/caesium-image-compressor

# 安装draw.io
scoop install extras/draw.io

# 安装pandoc
scoop install main/pandoc

# 安装upscayl
scoop install extras/upscayl

#安装typora1.7.6
scoop install extras/typora@1.7.6

#安装node.js
scoop install main/nodejs
```
## 卸载 Scoop
卸载命令会删除 Scoop 下的所有软件。
```powershell
scoop uninstall scoop
```