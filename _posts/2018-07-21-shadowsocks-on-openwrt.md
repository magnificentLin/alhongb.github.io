---
layout: post
title: OpenWrt Shadowsocks 安装&配置指南
categories: [Tutorial, Shadowsocks]
tags: [OpenWrt, Shadowsocks]
seo:
  date_modified: 2020-03-01 00:06:24 +0800
---

## 简介

本文先介绍了 OpenWrt Shadowsocks 及其配套软件的逻辑架构，然后详细说明了它们的安装、配置过程，最终搭建一个国内 IP 直连、其余 IP 走透明代理的翻墙路由器（即 CHNRoutes 黑名单模式）。

**提示：**要使用 GFWList 白名单模式（即仅被 GFW 封锁的流量走代理），请参阅：[《OpenWrt Shadowsocks GFWlist 配置教程》]({% post_url 2019-06-21-shadowsocks-on-openwrt-with-gfwlist %})。但博主推荐使用本文所述的 CHNRoutes 黑名单规则，能够避免部分 GFWList 遗漏的网站无法访问，并且加速外网访问速度（要求 Shadowsocks Server 的网络性能比较好）。

## 软件逻辑架构

Shadowsocks for OpenWrt 是基于 shadowsocks-libev 移植的，包含 **ss-local、ss-redir** 和 **ss-tunnel** 三个组件。此外，一个完整的 Shadowsocks 透明代理解决方案还包括 ChinaDNS 和远程 Shadowsocks 服务器等配套服务，整体逻辑架构图如下

![OpenWrt Shdowsocks 逻辑架构图](/assets/img/post/architecture-of-openwrt-shadowsocks-transparent-proxy.png)

其中，**ss-redir** 负责将 OpenWrt 的 TCP/UDP 出口流量透明地转发至境外 shadowsocks 代理服务器；**ss-local** 是本地 SOCKS5 代理服务器，可额外地为浏览器等客户端应用提供 SOCKS5 代理服务；**Dnsmaq** 是 OpenWrt 的默认 DNS 转发服务，本方案中负责接收来自局域网的 DNS 请求后转发给 ChinaDNS 处理；**ChinaDNS** 是一个开源的防 DNS 污染解决方案，它通过 **ss-tunnel** 转发 DNS 请求到墙外服务器，从而获取无污染的解析结果。

## **Step 1 - 安装 Shadowsocks**

先安装 shadowsocks-libev  `UDP-Relay` （UDP 转发）功能的依赖包 `iptables-mod-tproxy`

```sh
opkg update
opkg install iptables-mod-tproxy
```

由于 OpenWrt 内建的 wget 不支持 TLS，无法下载 HTTPS 网站上的软件包，因此还要安装好整版的 wget 和 CA 证书软件，前者负责下载链接，后者提供 HTTPS 连接所需的根证书：

```sh
opkg install wget ca-certificates
```

最后，正式开始安装 `shadowsocks-libev` 以及 `luci-app-shadowsocks`。本文提供项目作者最新预编译版本和软件源两种安装方式，读者可根据本地环境不同而择一：

### 下载编译版本安装

前往作者 github 主页获取最新的预编译版本 [shadowsocks-libev](https://github.com/shadowsocks/openwrt-shadowsocks/releases) 和 [luci-app-shadowsocks](https://github.com/shadowsocks/luci-app-shadowsocks/releases) UI 界面。注意，选择 `current` 目录下匹配你硬件架构的版本。使用以下命令查询你的路由器硬件架构，例如博主是 `x86_64`。

```sh
opkg print-architecture
```

获取链接后，下载并安装上述软件包

```sh
cd /tmp
wget https://xxx.ipk
opkg install xxx.ipk
```

### 用软件源安装

添加软件源公钥

```sh
wget http://openwrt-dist.sourceforge.net/packages/openwrt-dist.pub
opkg-key add openwrt-dist.pub
```

添加软件源到配置文件，注意替换 `x86_64` 为你的硬件架构（查询方法见上文）

```sh
vim /etc/opkg/customfeeds.conf

src/gz openwrt_dist http://openwrt-dist.sourceforge.net/packages/base/x86_64 src/gz openwrt_dist_luci http://openwrt-dist.sourceforge.net/packages/luci
```

安装 shadowsocks-libev、luci-app-shadowsocks

```sh
opkg update
opkg install shadowsocks-libev
opkg install luci-app-shadowsocks
```

## **Step 2 - 安装 ChinaDNS**

国内运营商网络 DNS 污染严重，导致大量境外域名无法正确解析，而 shadowsocks-libev 本身并没有解决 DNS 污染问题，需要配合 ChinaDNS 来解决此问题，其解决 DNS 污染的思路如下：

> ChinaDNS 分国内 DNS 和可信 DNS。ChinaDNS 会同时向国内 DNS 和可信 DNS 发请求，如果可信 DNS 先返回，则采用可信 DNS 的数据；如果国内 DNS 先返回，又分两种情况，返回的数据是国内的 IP, 则采用，否则丢弃并转而采用可信 DNS 的结果。
> 

安装方法同样有两种：

### 下载预编译版本安装

前往项目主页获取最新的预编译版本 [ChinaDNS](https://github.com/aa65535/openwrt-chinadns/releases) （选择 `current` 目录下特定硬件平台版本） 和 [chinadns-luci-app](https://github.com/aa65535/openwrt-dist-luci/releases) 并安装：

```sh
cd /tmp
wget https://xxx.ipk
opkg install xxx.ipk
```

### 用软件源安装

如果先前安装 shadowsocks-libev 时已添加了 openwrt-dist 源 ，可直接命令行安装 ChinaDNS

```sh
opkg install ChinaDNS
opkg install luci-app-chinadns
```

立即更新 ChinaDNS 的国内 IP 路由表 `/etc/chinadns_chnroute.txt`，这里分别提供 apnic 和国内的 [ipip.net](https://github.com/17mon/china_ip_list) 整理的路由表更新命令，推荐注重性能的使用后者：

- apnic

```sh
wget -O /tmp/delegated-apnic-latest 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' && awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' /tmp/delegated-apnic-latest > /etc/chinadns_chnroute.txt
```

- ipip.net

```
wget --no-check-certificate https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt -O /tmp/china_ip_list.txt && mv /tmp/china_ip_list.txt /etc/chinadns_chnroute.txt
```

编辑 crontab 任务计划，每周一凌晨 3 点更新 `chinadns_chnroute.txt`，

```sh
crontab -e

## For apnic
0 3 * * 1    wget http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest -O /tmp/delegated-apnic-latest && awk -F\| '/CN\|ipv4/ { printf("%s/%d\n", $4, 32-log($5)/log(2)) }' /tmp/delegated-apnic-latest > /etc/chinadns_chnroute.txt

## For ipip.net
0 3 * * 1    wget --no-check-certificate https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt -O /tmp/china_ip_list.txt && mv /tmp/china_ip_list.txt /etc/chinadns_chnroute.txt

/etc/init.d/cron start
/etc/init.d/cron enable
```

验证 crontab 任务是否正确执行

```sh
logread | grep crond
```

**注意：**不要直接使用 `vim` 编辑 `/etc/crontabs/root` 文件；本文提供的更新命令已经过优化，网络连接失败的情况下不会覆写错误信息到 `/etc/chinadns_chnroute.txt` 避免造成异常。

安装好后如图所示（忽略 vlmcsd ）：

![](/assets/img/post/shadowsocks-chinadns.png)

## **Step 3 - 配置 Shadowsocks**

- Servers Manage

管理 Shadowsocks 服务器节点。按照实际情况填写你的 Shadowsocks 代理服务器信息即可。

**注意**：如果要开启 `TCP Fast Open` 选项，需要修改 `sysctl.conf` 添加一行net.ipv4.tcp_fastopen = 3，然后使之生效。命令如下

```sh
echo "net.ipv4.tcp_fastopen = 3" >> /etc/sysctl.conf
sysctl -p
```

- General Settings

使能 `Transparent Proxy` 和 `Port Forward`，其中 `UDP-Relay` 是 UDP 转发功能，这里要将其开启，其余配置项保持默认即可。

- Access Control

Shadowsocks 的访问规则控制。一种合适规则是国外网站走 Shadowsocks 代理而国内网站直连，这样通常还可以加速国外网站，配置如下：

`Bypassed IP List` 选择 `ChinaDNS CHNRoute`

## **Step 4 - 配置 ChinaDNS**

勾选 `Also filter results inside China from foreign DNS servers`、将上游 DNS 修改为

`114.114.114.114,127.0.0.1:5300`

其中前者用于解析国内域名，建议使用运营商提供的DNS地址（注：因为有时114DNS速度慢于远程DNS，特别是远程DNS转发性能特别好时，而运营商DNS则一般不会发生这种情况。通过查看 /etc/resolv.conf 得到本地运营商DNS）；后者为 `ss-tunnel` 提供的 DNS 端口转发服务，负责DNS的远程服务器解析。最后，启动 ChinaDNS。

如图所示

![ChinaDNS 配置示例](/assets/img/post/chinadns-configuration.png) 

## **Step 5 - 配置 Dnsmasq**

OpenWrt 管理面 `Network` -> `DHCP and DNS`

`DNS forwardings` 修改为 `127.0.0.1#5353` 即 ChinaDNS 监听的端口；勾选 `Ignore resolve file`

提示：`Ignore resolve file` 指的是忽略 `/etc/resolv.conf` 中的 DNS ，即 WAN 口的 DNS，默认是电信运营商分配的 DNS；要恢复默认选项（取消勾选），在 `Resolve file` 一栏填入 `/tmp/resolv.conf.auto`

## Step 6 - 让路由器自身也翻墙（可选）

到此步骤为止，之前的配置已足以让接入局域网的设备正常翻墙，但为了让路由器自身发起的连接也能够翻墙成功，需要将 WAN 口默认使用的运营商 DNS 修改为 ChinaDNS

如图所示，在 `Network - Interfaces - WAN - Advanced Settings` 中去掉 `Use DNS servers advertised by peer` ，并在配置栏中填入 `127.0.0.1`

![OpenWrt WAN DNS Configuration](/assets/img/post/wan-advanced-settings.jpg)

至此，一切准备就绪，Enjoy yourself! :)


## 常见问题处理

- Shadowsocks 无法启动

重启路由器。如果仍未解决，使用 `logread` 命令查看异常日志
