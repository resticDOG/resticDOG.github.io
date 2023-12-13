---
title: 让Powershell更好用, Windows下目前最好用的终端工具-Windows Terminal折腾记
date: 2021-06-29
tags:
  - window terminal
  - 主题
  - 终端
categories:
  - tools
---

## 装 X

熟话说：“~~漂亮~~（ZB）是第一生产力”，既然都是黑框框，你希望是这样的黑框框：

![Axas2bnNV6DJLvQ.png](https://img.linkzz.eu.org/main/images/2023/08/d74740637b6c4ec89c473e4b2081161e.png)

还是这样的：

![EnDZ95XfQLuhUW2.png](https://img.linkzz.eu.org/main/images/2023/08/79f19ee09b82f04fa806e0479ba36e01.png)

又或者是这样的：

![HIzt8jAevWsrxSq.png](https://img.linkzz.eu.org/main/images/2023/08/bd04dd192addebb38e622a09f28201cd.png)

不！ 这并不是我想要的，每天要面对的黑框框。除非审美有问题，要不然谁不喜欢下面这个：

![7C5BgHtISyKbjiz.png](https://img.linkzz.eu.org/main/images/2023/08/c46cc9e6222af7625a6a276b9411d747.png)

## 如何装 X

到这里可能有些不明所以的小伙伴就要问了：“所以你这说的都是啥，没明白你装在哪里？”

## Windows Terminal

我们都知道，全球最大装机量的操作系统是 Windows，然而这样一个系统却有一个又丑有难用的终端-命令提示符，这使得很多开发者退而选择 linux 发行版进行开发，当然他们选择 linux 肯定还有其他原因，微软估计自己人都在吐槽自己的开发工具有多难用，所以陆续开发了 linux 子系统、Windows Terminal 等开发者友好的工具，这才使得我在 Windows 环境也能有不错的开发体验。在一众使用 cmd、powershell、git-bash 原生终端的小伙伴中成为焦点。

说了那么多废话，下面我们还是从实践中来探索一下 Windows Terminal 吧。

### 安装

打开 Microsoft Store 搜索 Windows Terminal 安装即可。

![1Uat5BJD6mNuCoH.png](https://img.linkzz.eu.org/main/images/2023/08/302813bf5018525af27b211e2652c763.png)

### 配置主题

最新版的 Windows Terminal 具有 GUI 配置界面，你可以在设置选项卡进行 GUI 的配置，如果不喜欢 GUI 界面配置，你还可以使用 JSON 配置的方式，点击打开 JSON 文件即可。

打开[windowsterminalthemes](https://windowsterminalthemes.dev/)，选择一个你喜欢的主题，然后复制到 JSON 文件 schemes 数组中， 如：

```JSON
{
  ...
  "profiles":
    {
      "defaults": {},
      "list":
      [
          {
              "acrylicOpacity": 0.5,        // 透明度
              "antialiasingMode": "cleartype",  // 字体渲染方式
              "backgroundImage": null,          // 背景图
              "backgroundImageOpacity": 0.14999999999999999,  // 背景图透明度
              "backgroundImageStretchMode": "uniformToFill",  // 背景图填充方式
              "colorScheme": "Monokai Vivid",     // 颜色主题
              "commandline": "powershell.exe -NoLogo",  // 执行的命令
              "fontFace": "JetBrainsMono Nerd Font",    // 字体
              "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",  // guid 可通过powershell的New-Guid生成
              "hidden": false,  // 是否隐藏标题
              "icon": "⚽",    // icon， 可使用emoji
              "name": "Windows PowerShell",  // 标题
              "useAcrylic": true     // 是否使用毛玻璃效果
          },
          {
              "commandline": "cmd.exe",
              "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
              "hidden": false,
              "name": "cmd"
          },
          {
              "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
              "hidden": false,
              "name": "Azure Cloud Shell",
              "source": "Windows.Terminal.Azure"
          },
          {
              "colorScheme": "Monokai Vivid",
              "commandline": "wsl",
              "fontFace": "JetBrainsMono Nerd Font",
              "guid": "{07b52e3e-de2c-5db4-bd2d-ba144ed6c273}",
              "hidden": false,
              "icon": "🐧",
              "name": "Ubuntu-20.04",
              "source": "Windows.Terminal.Wsl"
          }
        ]
  },
  "schemes":
    [
        {
            "background": "#0C0C0C",
            "black": "#0C0C0C",
            "blue": "#0037DA",
            "brightBlack": "#CC98E3",
            "brightBlue": "#3B78FF",
            "brightCyan": "#61D6D6",
            "brightGreen": "#16C60C",
            "brightPurple": "#B4009E",
            "brightRed": "#E74856",
            "brightWhite": "#F2F2F2",
            "brightYellow": "#F9F1A5",
            "cursorColor": "#FFFFFF",
            "cyan": "#3A96DD",
            "foreground": "#CCCCCC",
            "green": "#13A10E",
            "name": "Campbell",
            "purple": "#881798",
            "red": "#C50F1F",
            "selectionBackground": "#FFFFFF",
            "white": "#CCCCCC",
            "yellow": "#C19C00"
        },
        {
            "background": "#012456",
            "black": "#0C0C0C",
            "blue": "#0037DA",
            "brightBlack": "#767676",
            "brightBlue": "#3B78FF",
            "brightCyan": "#61D6D6",
            "brightGreen": "#16C60C",
            "brightPurple": "#B4009E",
            "brightRed": "#E74856",
            "brightWhite": "#F2F2F2",
            "brightYellow": "#F9F1A5",
            "cursorColor": "#FFFFFF",
            "cyan": "#3A96DD",
            "foreground": "#CCCCCC",
            "green": "#13A10E",
            "name": "Campbell Powershell",
            "purple": "#881798",
            "red": "#C50F1F",
            "selectionBackground": "#FFFFFF",
            "white": "#CCCCCC",
            "yellow": "#C19C00"
        },
        {
            "background": "#121212",
            "black": "#121212",
            "blue": "#0443FF",
            "brightBlack": "#ABA78B",
            "brightBlue": "#0443FF",
            "brightCyan": "#51CEFF",
            "brightGreen": "#B1E05F",
            "brightPurple": "#F200F6",
            "brightRed": "#F6669D",
            "brightWhite": "#FFFFFF",
            "brightYellow": "#FFF26D",
            "cursorColor": "#FB0007",
            "cyan": "#01B6ED",
            "foreground": "#F9F9F9",
            "green": "#98E123",
            "name": "Monokai Vivid",
            "purple": "#F800F8",
            "red": "#FA2934",
            "selectionBackground": "#FFFFFF",
            "white": "#FFFFFF",
            "yellow": "#FFF30A"
        }
    ]
}
```

### 安装 Oh-My-Posh

常用 linux 的人应该对 Oh-My-Zsh 不陌生，Oh-My-Zsh 加上众多的 plugin，使终端使用的舒适度提升了很多，[Oh-My-Posh](https://ohmyposh.dev/)即是对 powershell 做 prompt 美化的一个引擎。

管理员身份打开 powershell, 执行：

```powershell
Install-Module oh-my-posh -Scope CurrentUser
```

若提示不允许安装是因为 Windows 的脚本执行策略阻止的，需要修改策略，同样管理员运行：

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Confirm
```

安装好之后执`Import-Module oh-my-posh`导入模块，因为导入只在会话范围内有效，为了保证每次开启终端之前都导入模块，在 powershell 启动脚本中加入代码。使用编辑器打开$profile，我用的编辑器是 vscode。

```powershell
code $profile  # 用vscode打开powershell配置文件，没有这个文件会自己创建
```

`$profile`类似`.bashrc`，每次开启一个会话都会执行该脚本，适合将各种 alias 放到其中，文件中添加

```powershell
Import-Module oh-my-posh
```

### 选择一个合适的 prompt 主题

Oh-My-Posh内置了多个主题，其颜值和实用性我觉得够用了，使用`Get-PoshThemes`可预览所有主题。 ![4AK6Jey9VHMiYq1.png](https://img.linkzz.eu.org/main/images/2023/08/55f6ddd3f7ed717ef9af8cbaf3fd232c.png)

使用`Set-PoshPrompt -Theme $THEME`设置主题，选好主题之后也将设置主题的脚本写到 profile 中：

```powershell
Import-Module oh-my-posh
Set-PoshPrompt -Theme ys  # 我使用ys主题
```

### 选择一个等宽字体

也许你的主题会有些字符显示不出，比如这样的：

![hwkPcZmKvf7Ny9H.png](https://img.linkzz.eu.org/main/images/2023/08/59b2c1bd6146c4e5a8a962d7d0c43661.png)

这是因为字体不支持图标引起的，需要 Nerd字体方可解决这个问题。具体可参考这个项目: [Nerd Font](https://github.com/ryanoasis/nerd-fonts) 我是用的是[JetbrainsMono Nerd Patcher](https://raw.githubusercontent.com/ryanoasis/nerd-fonts/master/patched-fonts/JetBrainsMono/Ligatures/Regular/complete/JetBrains%20Mono%20Regular%20Nerd%20Font%20Complete%20Mono%20Windows%20Compatible.ttf)的字体。安装字体之后可在设置界面选择该字体：

![Oy3frwQ5lmtHGVI.png](https://img.linkzz.eu.org/main/images/2023/08/6aecea3b8fb073c02f6c35ed43e5b786.png)

### 其他一些设置

至于添加背景图，设置透明度等这些设置相信就不用我多说什么了，相信你也能配出一个高颜值的终端，最后贴出我的 JSON 设置和 profile 供大家参考。最后说一下一些主题的颜色配置在命令的参数如：`git --version`中`--version`可能会很难辨认，主要因为该颜色和背景色的对比度太低。

![oKnaA7dMel2kJuO.png](https://img.linkzz.eu.org/main/images/2023/08/387941d38aa7adad63924f1e1470e92f.png)

这个时候换一个主题当然是个办法，但如果你真很喜欢这个主题也不是没有解决方法，打开设置-配色方案，找到对比度很低的颜色，修改为相对背景色高一点的颜色即可，找不到颜色，相似的试一下总会使出来的。

![nIQTlyp4tAuWDwh.png](https://img.linkzz.eu.org/main/images/2023/08/05201ef4f180d0ebfedd9536a57a1e0c.png)

JSON 配置（省略了一些配置）：

```JSON
{
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "actions":
    [
        {
            "command":
            {
                "action": "copy",
                "singleLine": false
            },
            "keys": "ctrl+c"
        },
        {
            "command": "paste",
            "keys": "ctrl+v"
        },
        {
            "command": "find",
            "keys": "ctrl+shift+f"
        },
        {
            "command":
            {
                "action": "splitPane",
                "split": "auto",
                "splitMode": "duplicate"
            },
            "keys": "alt+shift+d"
        }
    ],
    "copyFormatting": "none",
    "copyOnSelect": false,
    "defaultProfile": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
    "profiles":
    {
        "defaults": {},
        "list":
        [
            {
                "acrylicOpacity": 0.5,
                "antialiasingMode": "cleartype",
                "backgroundImage": null,
                "backgroundImageOpacity": 0.14999999999999999,
                "backgroundImageStretchMode": "uniformToFill",
                "colorScheme": "Monokai Vivid",
                "commandline": "powershell.exe -NoLogo",
                "fontFace": "JetBrainsMono Nerd Font",
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "hidden": false,
                "icon": "\u26bd",
                "name": "Windows PowerShell",
                "useAcrylic": true
            },
            {
                "commandline": "cmd.exe",
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "hidden": false,
                "name": "\u547d\u4ee4\u63d0\u793a\u7b26"
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            },
            {
                "colorScheme": "Monokai Vivid",
                "commandline": "wsl",
                "fontFace": "JetBrainsMono Nerd Font",
                "guid": "{07b52e3e-de2c-5db4-bd2d-ba144ed6c273}",
                "hidden": false,
                "icon": "\ud83d\udc27",
                "name": "Ubuntu-20.04",
                "source": "Windows.Terminal.Wsl"
            }
        ]
    },
    "schemes":
    [
        {
            "background": "#121212",
            "black": "#121212",
            "blue": "#0443FF",
            "brightBlack": "#ABA78B",
            "brightBlue": "#0443FF",
            "brightCyan": "#51CEFF",
            "brightGreen": "#B1E05F",
            "brightPurple": "#F200F6",
            "brightRed": "#F6669D",
            "brightWhite": "#FFFFFF",
            "brightYellow": "#FFF26D",
            "cursorColor": "#FB0007",
            "cyan": "#01B6ED",
            "foreground": "#F9F9F9",
            "green": "#98E123",
            "name": "Monokai Vivid",
            "purple": "#F800F8",
            "red": "#FA2934",
            "selectionBackground": "#FFFFFF",
            "white": "#FFFFFF",
            "yellow": "#FFF30A"
        }
    ],
    "theme": "system"
}
```

profile:

```powershell
# some custome scripts

# import on-my-posh
Import-Module oh-my-posh

# setup oh-my-posh theme
Set-PoshPrompt -Theme ys

# alais scripts
. C:\Users\linkzz\ps-scripts\alais.ps1
```

## 结语

至此 Windows Terminal 美化就说完了，到这里你可以去给你的小伙伴们装 X 了，但这只算装 X 刚入门阶段， 要想更好的装 X，想要更多好用的终端功能，下一章我们继续。
