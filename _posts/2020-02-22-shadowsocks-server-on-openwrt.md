---
title: OpenWrt 安装 Shadowsocks Server
categories: [Tutorial, Shadowsocks]
tags: Shadowsocks
layout: post
seo:
  date_modified: 2020-03-02 22:39:56 +0800
---

在 OpenWrt 路由器上安装 server 版 Shadowsocks，使得客户端设备能够以 VPN 的形式远程连接到家庭局域网。

## Step 1 - 安装 shadowsocks-libev-sever

前往作者[项目主页](https://github.com/shadowsocks/openwrt-shadowsocks/releases)获取最新版本的 shadowsocks-libev-sever 并安装

## Step 2 - 创建配置文件

```sh
vim /etc/shadowsocks.json

{
   "server":["[::0]","0.0.0.0"],
    "server_port":8888,
    "password":"xxxxxxx",
    "timeout":60,
    "method":"aes-128-gcm",
    "fast_open":true
}
```

## Step 3 - 创建开机启动脚本

```sh
vim /etc/rc.local

## 在文件最后， exit 0 之前（如果有的话）加此行启动命令
ss-server -u -c /etc/shadowsocks.json &
```
