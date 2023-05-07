---
title: Ubuntu修改默认终端
date: 2020-09-19 21:59:47
tags: [Linux, 随笔]
keywords:
description: 在Ubuntu下修改默认终端为Debian终端。
---

在终端中输入如下命令即可，当然前提是安装了`Debian`的深度终端。

```  
gsettings set org.gnome.desktop.default-applications.terminal exec /usr/bin/deepin-terminal  

gsettings set org.gnome.desktop.default-applications.terminal exec-arg "-x"

```

