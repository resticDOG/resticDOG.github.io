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

## 装X

熟话说：“~~漂亮~~（ZB）是第一生产力”，既然都是黑框框，你希望是这样的黑框框：
![dos](https://i.loli.net/2021/07/01/Axas2bnNV6DJLvQ.png)
还是这样的：
![git-bash](https://i.loli.net/2021/07/01/EnDZ95XfQLuhUW2.png)
又或者是这样的：
![powershell](https://i.loli.net/2021/07/01/HIzt8jAevWsrxSq.png)
不！
这并不是我想要的，每天要面对的黑框框。除非我审美有问题， 要不然谁不喜欢下面这个：
![windows-terminal](https://i.loli.net/2021/07/01/7C5BgHtISyKbjiz.png)

## 如何装X

到这里可能有些不明所以的小伙伴就要问了：“所以你这说的都是啥，没明白你装在哪里？”

### Windows Terminal

我们都知道，全球最大装机量的操作系统是Windows，然而这样一个系统却有一个又丑有难用的终端-命令提示符，这使得很多开发者退而选择linux发行版进行开发，当然他们选择linux肯定还有其他原因，微软估计自己人都在吐槽自己的开发工具有多难用，所以陆续开发了linux子系统、Windows Terminal等开发者友好的工具，这才使得我在Windows环境也能有不错的开发体验。在一众使用cmd、powershell、git-bash原生终端的小伙伴中成为焦点。

说了那么多废话，下面我们还是从实践中来探索一下Windows Terminal吧。

#### 安装

打开Microsoft Store搜索Windows Terminal安装即可。
![microsoft-windows-terminal](https://i.loli.net/2021/07/01/1Uat5BJD6mNuCoH.png)

#### 配置主题

最新版的Windows Terminal具有GUI配置界面，你可以在设置选项卡进行GUI的配置，如果不喜欢GUI界面配置，你还可以使用JSON配置的方式，点击打开JSON文件即可。

打开[windowsterminalthemes](https://windowsterminalthemes.dev/)，选择一个你喜欢的主题， 然后复制到JSON文件schemes数组中， 如：

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

#### 安装Oh-My-Posh

常用linux的人应该对Oh-My-Zsh不陌生，Oh-My-Zsh加上众多的plugin，使终端使用的舒适度提升了很多，[Oh-My-Posh](https://ohmyposh.dev/)即是对powershell做prompt美化的一个引擎。

管理员身份打开powershell, 执行：

```powershell
Install-Module oh-my-posh -Scope CurrentUser
```

若提示不允许安装是因为Windows的脚本执行策略阻止的，需要修改策略，同样管理员运行：

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Confirm
```

安装好之后执`Import-Module oh-my-posh`导入模块，因为导入只在会话范围内有效，为了保证每次开启终端之前都导入模块，在powershell启动脚本中加入代码。使用编辑器打开$profile，我用的编辑器是vscode。

```powershell
code $profile  # 用vscode打开powershell配置文件，没有这个文件会自己创建
```

`$profile`类似`.bashrc`，每次开启一个会话都会执行该脚本，适合将各种alias放到其中，文件中添加

```powershell
Import-Module oh-my-posh
```

#### 选择一个合适的prompt主题

Oh-My-Posh内置了多个主题，其颜值和实用性我觉得够用了，使用`Get-PoshThemes`可预览所有主题。
![poshthemes](https://i.loli.net/2021/07/01/4AK6Jey9VHMiYq1.png)
使用`Set-PoshPrompt -Theme $THEME`设置主题，选好主题之后也将设置主题的脚本写到profile中：

```powershell
Import-Module oh-my-posh
Set-PoshPrompt -Theme ys  # 我使用ys主题
```

#### 选择一个等宽字体

也许你的主题会有些字符显示不出，比如这样的：
![font](https://i.loli.net/2021/07/01/hwkPcZmKvf7Ny9H.png)
这是因为字体不支持图标引起的，需要Nerd字体方可解决这个问题。具体可参考这个项目: [Nerd Font](https://github.com/ryanoasis/nerd-fonts)
我是用的是[JetbrainsMono Nerd Patcher](https://raw.githubusercontent.com/ryanoasis/nerd-fonts/master/patched-fonts/JetBrainsMono/Ligatures/Regular/complete/JetBrains%20Mono%20Regular%20Nerd%20Font%20Complete%20Mono%20Windows%20Compatible.ttf)的字体。安装字体之后可在设置界面选择该字体：
![setup-font](https://i.loli.net/2021/07/01/Oy3frwQ5lmtHGVI.png)

#### 其他一些设置

至于添加背景图，设置透明度等这些设置相信就不用我多说什么了，相信你也能配出一个高颜值的终端， 最后贴出我的JSON设置和profile供大家参考。
最后说一下一些主题的颜色配置在命令的参数如：`git --version`中`--version`可能会很难辨认，主要因为该颜色和背景色的对比度太低。
![param-color](https://i.loli.net/2021/07/01/oKnaA7dMel2kJuO.png)
这个时候换一个主题当然是个办法，但如果你真很喜欢这个主题也不是没有解决方法，打开设置-配色方案，找到对比度很低的颜色，修改为相对背景色高一点的颜色即可，找不到颜色，相似的试一下总会使出来的。
![theme-color-high](https://i.loli.net/2021/07/01/nIQTlyp4tAuWDwh.png)

JSON配置（省略了一些配置）：

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

至此Windows Terminal美化就说完了，到这里你可以去给你的小伙伴们装X了，但这只算装X刚入门阶段， 要想更好的装X，想要更多好用的终端功能，下一章我们继续。
