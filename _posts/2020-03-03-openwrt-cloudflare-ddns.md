---
title: OpenWrt Cloudflare DDNS
categories: [Tutorial, OpenWrt]
tags: [OpenWrt, DDNS]
layout: post
---

本文演示了如何在 OpenWrt 上设置 DDNS，接入 Cloudflare 提供的 DDNS 服务。

## 获取 API key

Cloudflare DDNS API 是 RESTful API，调用时必须使用 API key。前往 [My account](https://dash.cloudflare.com/profile/api-tokens) 页面获取自己的 API key即可。

## 安装 ddns 软件包

需要安装 `ddns-scripts` 和 `ddns-scripts_cloudfare.com-v4` 两个人软件包

```sh
opkg install ddns-scripts ddns-scripts_cloudfare.com-v4
```

## 配置 ddns

在 `Dynamic DNS` LuCI 界面中，新建一个 DDNS 配置项，内容如下：

- 勾选 `Enabled`
- `Lookup HostName`: 要执行 DDNS 的完全限定域名（FQDN），例如 `subdomain.example.com`
- `IP address version`: 勾选 `IPv4-Address`
- `DDNS Service provider`: 选择 `cloudfare.com-v4`
- `Domain`: 按 `主机名@域名` 的格式填写，例如 `subdomain@example.com`
- `Password/密码`：填写先前获取的 Cloudflare API key
- `Optional Parameter`: 填写 `"proxied":false`。由于默认地 Cloudflare API 会开启 `proxied` 功能（会在 Cloudflare DNS 页面点亮域名黄色的云朵）,而我们家用 DDNS 一般只是为了获取 IP 地址，所以要关闭这个选项转变为 `DNS only` (即让黄色的云朵变灰)，否则你的域名会自动开启 `proxied` 功能

以下选项表示要求使用 HTTPS 安全通道访问 Cloudflare，出于安全考量，建议打开

- 勾选 `Use HTTP Secure`
- `Path to CA-Certificate`: 填写 `/etc/ssl/certs` 注：OpenWrt 默认未预置 CA 根证书，使用如下命令安装

```sh
opkg install ca-certificates
```

最后，点击 `Save & Apply` 观察 DDNS 是否生效即可。
