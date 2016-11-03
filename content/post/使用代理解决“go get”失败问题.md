+++
comments = true
slug = ""
date = "2016-10-30T21:46:36+08:00"
title = "使用代理解决‘go get’失败问题"
toc = false
draft = false

+++

## 问题描述
自从有了自己的ss服务器之后，基本再没有为翻墙而烦恼。然而，最近却遇到了一个奇怪的现象，使用`go get`安装library的时候，有一些包会下不下来，报出类似`unrecognized import path dial tcp i/o timeout`这样的错误，上网查了一下，是`go get`命令不会自动走socks代理，即使配置全局模式也不行。会出现这种问题的包大都是在“golang.org”路径下，GFW不知道为啥脑抽连golang.org都要封。之前使用VPN(PPTP)翻墙的时候，不会遇到这个问题，因为PPTP属于数据链路层，而ss属于传输层。

## 解决方案
实际上，`go get`命令不过是`git clone`命令加上`go install`命令的结合（通过搜索错误类型时，猜测git clone 内部是使用curl下载的，没有具体求证）。所以确切地说，只需要解决git不能走socks代理的问题。上网查询到git可以配置http代理，可是我使用的是socks代理，试了一下直接使用socks代理的本地地址，果然是不行的。突然想到之前用于给chromebook激活时用到的“privoxy”，通过http代理转socks代理，终于OK了。具体些，就是开启privoxy监听一个本地地址，配置中，设置成以socks方式转发到本地的socks地址，然后在`go get`命令前指定使用prixoxy监听的地址，类似`http_proxy=http://127.0.0.1:xxxx https_proxy=https://127.0.0.1:xxxx go get`，大功告成。