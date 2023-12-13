---
title: 基于AutoHotKey的windows平台Vim输入法切换方案
date: 2023-09-05
description: "基于AutoHotKey的windows平台Vim输入法切换方案"

tags:
  - AutoHotKey
  - Vim
categories:
  - Vim
  - AutoHotKey
lightgallery: true

toc:
  auto: true
---

众所周知，在中文环境下使用 vim 一直会有中英文输入法切换的烦恼，社区也有一些方案解决，如 AutoHotKey 脚本 `im-select` 插件等，今天提供一种基于屏幕颜色的方案来实现按下 `ESC` 键切换输 入法的方案，下面是效果展示：

![output.gif](https://img.linkzz.eu.org/main/images/2023/09/6521c2a2ace052053b9608ebea643f6f.gif)

## Window Spy

我目前安装的是[AutoHotKey v2.0.2](https://www.autohotkey.com/docs/v2/)版本，使用其提供的 Window Spy 功能，我们可以方便的查看各个窗口的信息，鼠标的信息等，今天我们要实现的版本思路就是基于判断屏幕指定坐标处的十六进制颜色来判断当前是处于什么输入法环境下。

![image.png](https://img.linkzz.eu.org/main/images/2023/09/48d21a2cade00018cbcec330defb00da.png)

如上图，因为截图无法截取鼠标位置，我的鼠标置于图中“中”字的下部分，切换到英文的时候为大写字母"A"，屏幕相同的位置是没有“中”字下部分的颜色的，而 Windows 系统输入法状态指示的图标相对来说是比较固定的，故以此来判断当前输入法的环境是可行的，看官自己设置的时候使用 window spy 来获取这个坐标即可。

## AutoHotKey 脚本

既然有了方案就可以着手实现了，我们使用 V2 版本的语法来完成，并将键盘上不常用的 `CapsLock` 键映射为 `ESC` 键，下面是完整的脚本：

```autohotkey
#Requires AutoHotkey v2.0

; 获取坐标的模式，相对于屏幕
CoordMode "Pixel", "Screen"
CoordMode "Mouse", "Screen"

; 应用vim模式的窗口组
GroupAdd  "VimMode", "ahk_exe WindowsTerminal.exe"
GroupAdd  "VimMode", "ahk_exe Code.exe"
GroupAdd  "VimMode", "ahk_exe Obsidian.exe"

; 退出vim模式
VimEsc() {
  Send "{Esc}"
  if (PixelGetColor(1985, 1422) = "0x464646")
    Send "{Shift}"
}


#HotIf WinActive("ahk_group VimMode")
ESC::VimEsc()
CapsLock::VimEsc()
#HotIf
```

以上代码中将需要生效的应用添加到 `VimMode` 组，如果有更多需要应用 Vim 模式的应用，通过 window spy 获取其 ahk_exe，通过`GroupAdd "VimMode", "ahk_exe {name}"` 将其加入组中即可，十六进制颜色和坐标也根据需要修改为你自己的，Send 后面的内容也要修改你输入法的中英文切换的快捷键，在这里我的是 `Shift` 键，之后运行脚本即可实现“按下 `Esc` 或者`CapsLock` 自动切换为英文输入法” 的功能。
