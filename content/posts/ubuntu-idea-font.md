---
title: 拯救你的idea,解决ubuntu下idea字体发虚 
date: 2019-05-03 22:36:15
tags: 
- IDEA
- ubuntu 
- 字体
categories: 
- linux
---

### idea在ubuntu下的字体表现

idea这么强大的ide相信大家都有目共睹，但是最近在ubuntu下安装idea之后却发现字体表现还不如windows，要知道windows的字体渲染可是不如linux的，在高分屏下window的字体会出现明显锯齿，而linux就很平滑，虽然这个和idea没关系，因为idea是用java开发的，字体渲染不是用的系统的字体渲染引擎。查阅资料也发现对于idea的字体渲染问题，intelliJ是有优化过的，然而我ubuntu下的idea看起来却是这样的：
<!-- more -->
![字体发虚](https://i.imgur.com/YkDKWyj.jpg)

### idea环境

我的ubuntu是18.04LTS，屏幕是23寸1920*1080，idea设置如下：
![主题设置](https://i.imgur.com/EzED46g.png)
主题方面选择了idea内部提供的暗色主题Darcula
编辑界面主题是在[idea主题样式网站](https://www.riaway.com/)下载的：
![编辑界面主题](https://i.imgur.com/3yHaDWq.png)
英文字体是Adobe开源的source code pro,在[github仓库](https://github.com/adobe-fonts/source-code-pro)中提供下载。中文字体是开源字体[文泉驿微米黑](http://wenq.org/wqy2/index.cgi?MicroHei)

### 更换系统主题

字体看起来发虚只是在暗色主题中才会发生，所以我们采取曲线救国策略，换用itelliJ亮色主题，
然后结合一张暗色背景图来实现暗色主题的效果。
![主题](https://i.imgur.com/kOYK47F.png)
主题选择intellJ
![背景图](https://i.imgur.com/C72WEg4.png)
选择一张背景图设置透明度
![编辑界面主题](https://i.imgur.com/eEdwc6X.png)
编辑界面主题选择一张亮色的主题，字体不变

### 成果

完成以上设置之后来看看成果
![成果](https://i.imgur.com/FA32hdy.jpg)
字体看起来平滑多了，好了，安心编码吧 :)
