---
title: openmediavault NAS 搭建指南
categories: [Tutorial, NAS]
tags: [openmediavault]
layout: post
seo:
  date_modified: 2020-04-08 22:42:40 +0800
---

本文详细记录了在 openmediavault 上搭建私人 NAS 的过程，包括：安装配置 openmediavault、Docker；部署 Transmission BT 工具、Nextcloud 网盘等容器；配置 HTTP/HTTPS 反向代理和 Let's Encrypt 证书，最终实现个人 NAS 的搭建。

## 硬件选择

博主用过的 DIY NAS 硬件有小马 V5（已退役）和蜗牛星际，这里提一些硬件选购建议，供读者参考：

- 专用设备

NAS 的核心功能应当是可靠的数据存储，长期稳定运行是一大要素，因此不建议使用虚拟化技术（一设备多用途）、树莓派等非专用设备来构建你的 NAS。

- 低功耗

低功耗不仅意味着绿色清洁，还带来更好的散热性能，这些都是 NAS 设备长期运行的基础

- 盘位至少 3，最优 4

个人认为兼顾数据安全和丰富应用的硬盘布置策略是：占用 2 盘位的 `RAID 1`(mirror) + 1 盘位单盘（或 2 盘位 `RAID 0`），前者用于存储个人数据或稀缺资源，后者用于 BT、电影分享等数据容易重新下载的场景。因此满足此策略的 4 盘位硬件就足矣，至于超过 4 盘位的，个人觉得不必要。

## NAS 系统选择

- Synology DSM

适合无技术背景或希望开箱即用的用户。博主不选择的原因：

1.相对臃肿，不够简洁
2.每块磁盘上都会安装 DSM，包括存储安装的软件及其数据，导致用户存储和操作系统耦合，尤其是频繁的读写影响每块硬盘的休眠功能。

- FreeNAS

对硬件要求比较高，尤其是内存最低要求 8G，其未来的目标用户应该主要是企业，不选。

- openmediavault

基于 Debian Linux，开源免费。openmediavault 面向的就是家庭用户和小型办公环境，是对 Linux 熟悉又追求最小化安装
的人的首选。

另外，所有的 NAS 系统都有物理机裸装和运行在 ESXi 等虚拟化平台上两种区分，但考虑到虚拟机对 S.M.A.R.T、磁盘休眠等需要硬件直通的特性支持不好，而这些功能是 NAS 长期稳定、低功耗运行的核心，因此强烈建议不要使用虚拟化安装 NAS。

## 安装 openmediavault

openmediavault 基于 Debian，因此安装过程与绝大部分 Linux 发行版没什么两样：先在[官网](https://www.openmediavault.org/?page_id=77)下载 ISO 文件，解压到 U 做成启动盘，最后引导设备启动到安装程序完成安装。几个注意事项：

- 在安装界面执行磁盘重新分区时，可能会报无法安装系统文件的错误，忽略错误重启一次即可
- 网站提供的 ISO 镜像最新版本是 5.0.5，但安装完成后会自动升级到最新版本
- 安装过程确保联网更新，避免出现网卡无法驱动的问题。更新源选择国内的节点，如清华，否则速度极慢
- 安装过程语言选项可选择英文或中文，影响 Debian 和 openmediavault 的语言。建议选英文，因为 openmediavault 的时区、语言安装完成后很容易通过其界面更改

## 创建共享文件夹

openmediavault 成功运行后，就可以用其 Web 图形界面组建 RAID 和创建可被外部访问的`共享文件夹`了，基本流程是：

```
清除磁盘（可选）- 组建 RAID（可选）- 创建文件系统 - 挂载文件系统 - 添加共享文件夹 - 打开文件共享服务（可选）
```

注意：默认的 `admin` 管理员用户不能用于访问共享文件夹，需要新建一个普通用户；所有共享文件访问都应当通过 openmediavault `共享文件夹`机制：主机外使用各种基于网络的文件共享服务，主机内使用 `/sharedfolders/yourfolder` 路径来访问共享文件夹。

## 安装 Docker 环境

### 安装 Docker

为了运行 Nextcloud（云盘）、Transmission（BT 下载）等软件，需要安装和配置 Docker 运行环境。

首先按其官网的方法安装三方扩展插件库 [omv-extras](http://omv-extras.org/) 

```sh
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

安装完毕后，直接在新增的 `omv-extras` 界面中点击按钮安装 Docker 即可

### 安装 Portainer

Portainer 是一个轻量级的 Docker 图形化管理工具，可以用来管理宿主机和 Docker Swarm 集群。Portainer 本身也是以容器运行，因此安装过程就是创建和运行一个容器。

omv-extras 也提供了 Portainer 的安装界面，直接用它来安装即可。

安装完成后访问 *openmediavualt_host_ip*:9000 即可访问其 Web 界面，首次登录需要创建一个管理员用户，然后会让你选择要连接的 Docker 环境，这里我们选择 `Local`，即此前安装的 Docker。

## 部署 NAS 常用应用容器

### docker container 命令基础

- Docker 默认以 root 运行容器进程，除非 `docker container` 命令指定了 `--user` 参数
- `docker container run` 命令是创建并运行一个容器，`docker container create` 则仅创建容器而不运行。要运行已有的容器，使用 `docker container start` 命令
- `--rm` 参数是容器运行结束后自动删除该容器文件，由于默认情况下 Docker 每次启动容器都会新创建一个对应的容器文件， 对于仅一次性运行容器的场景，`--rm` 参数就会很有用
- `-v` 参数说明：

Docker 支持 3 种文件系统实现：`volume`，`bind mount` 和 `tmpfs mount`，它们分别用不同的方式将容器内的文件/目录和宿主的文件/目录关联，使容器能够访问宿主的文件系统/内存文件系统。 其中 `volume` 是 Docker 自行管理的文件/目录，也是最易使用的：通过 `-v` 参数指定一个唯一的 volume name 以及要关联到的容器内文件/目录即可。`volume` 由 Docker 管理，使用时可以是已经存在或未存在的任一个，当不存在时 Docker 会负责创建；`bind mount` 是将容器内的目录/文件绑定到宿主上的已有的目录或文件，用法是和 `volume` 类似，但 `-v` 参数指定的一定是宿主机上某个相对路径或绝对路径。

![docker volumes](https://docs.docker.com/storage/images/types-of-mounts-volume.png)

- `-e` 参数，增加一个供容器使用的环境变量

- `PUID/PGID`，是 [LinuxServer.io](https://www.linuxserver.io/) 组织提供的镜像特有的实用功能，作为环境变量指定，以指定容器进程运行所用的 UID/GID

### 部署 Transmission 容器

Transmission 用来下载 BT（支持磁力链接）非常不错，支持 Web 界面和包含认证的 RPC 控制，我们选择 linuxserver 提供的镜像，[Docker Hub - linuxserver/transmission](https://hub.docker.com/r/linuxserver/transmission/)

容器安装是用 Portainer，具体操作步骤是：

1. 进入 `Volumes` - `Add volume` 页面创建 Transmission 容器配置数据存储专用 `volume`
    - 填写 `Name`，例如 `transmission_config`
    - 点击 `Create the volume` 创建该 `volume`
2. 进入 `Containers` - `Add container` 页面配置容器参数
    - 填写任意的 `Name`，填写 `Image` 为 `linuxserver/transmission`
    - `Manual network port publishing` 中点击 `publish a new network port`，按 linuxserver/transmission 在 Docker Hub 页面的要求依次添加端口映射
    - `Volumes` - `Volume mapping` 选项中将专用于存储配置数据的 `Volume` 即 `transmission_config`，绑定到 `/config` 和 `/watch`；`Bind` 形式绑定 `/downloads` 到你指定的 NAS 共享目录。
    - `Env` - `Environment variables` 中添加 `PUID`、`PGID` 两个环境变量，可以分别填写 `1000` 和 `100` 即你的 openmediavault 使用用户和 `users` 组
    - `Restart policy` 中选择 `Unless stopped`
    - 最后点击 `Deploy the container` 完成容器部署

### 部署 Nextcloud 容器

Nextcloud 作为云盘软件，实际上主要的文件管理功能完全可以使用 openmediavault 提供的 Samba/NFS 替代，但是如果要在 Internet 上分享文件，后者就十分乏力了。

我同样选择 linuxserver 提供的镜像，[Docker Hub - linuxserver/nextcloud](https://hub.docker.com/r/linuxserver/nextcloud)。之所以不选择官方镜像，是因为其不支持设置容器进程的 UID/GID，无法控制容器进程的读写权限。

Nextcloud 容器运行起来后，还要编辑一下它的配置文件，将域名修正为你自己的实际域名，我的例子是 `nextcloud.linhongbo.com`

```sh
vim /var/lib/docker/volumes/nextcloud_config/_data/www/nextcloud/config/config.php

<?php
$CONFIG = array (
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'datadirectory' => '/data',
  'instanceid' => 'oc2sfcyt03u3',
  'passwordsalt' => '1234567890abcdefg...',
  'secret' => '1234567890abcdefg...',
  'trusted_domains' =>
  array (
          0 => 'openmediavault',
          1 => 'nextcloud.linhongbo.com',
  ),
  'dbtype' => 'sqlite3',
  'version' => '18.0.1.3',
  'overwrite.cli.url' => 'https://nextcloud.linhongbo.com:8443',
  'installed' => true,
);
```

### 部署 Emby Server 容器

相比 Plex 博主更喜欢 Emby。镜像地址：[Docker Hub - emby/embyserver](https://hub.docker.com/r/emby/embyserver/)

由于 Emby Server 一般只需要访问媒体文件，出于安全考量，我们新建一个仅可访问媒体目录的用户，专门用来运行 Emby Server 容器

新建名为 `mediaonly` 的用户，并查询其 UID

```sh
sudo useradd mediaonly -g 100
id mediaonly
```

这里 `-g` 将 GID 配置为 `100`，即 openmediavault 默认的 `users` 组，使 `mediaonly` 能够访问共享文件夹

Portainer 修改 `UID` 环境变量，设置为查询到的 `mediaonly` 用户 UID，博主是 `1001`

上述做法的好处是你可以在 openmediavault 共享文件夹的 ACL 中配置 `mediaonly` 访问权限，限制 Emby Server 对私人数据目录的访问。(我的做法是去掉私有数据目录默认的 `users` 组访问权限，仅主用户有权访问，其中明确开放给 Emby Server 访问的媒体数据目录除外)

### 使用域名访问容器

默认情况下，我们会用 `IP:端口` 方式访问容器提供的 Web 服务，并且用端口号来区分不同的服务，这在容器数量较多时显得非常不优雅。

有两种解决方案，分别适用不同场景：

1. 如果你有公开域名并希望容器能够被公网访问，可以在域名 DNS 服务提供商中配置新的容器服务域名，并解析到可访问容器的公网 IP（这通常需要宽带运营商提供公网 IP，并且你已[配置好 DDNS](https://linhongbo.com/posts/openwrt-cloudflare-ddns/)）
2. 如果你没有公开域名或仅在局域网访问容器服务，可以配置本地域名。具体方法不一，博主是在 OpenWrt 路由器中修改 `/etc/hosts` 文件完成的，形如：

```
192.168.0.2 portainer.linhongbo.com
192.168.0.2 transmission.linhongbo.com
192.168.0.2 openmediavault.linhongbo.com
192.168.0.2 openwrt.linhongbo.com
192.168.0.2 emby.linhongbo.com
```
## 启用 HTTPS

上面我已经运行了 3 个需要身份认证的 Web 服务（包括 Portainer），其中 Nextcloud 甚至还需要在公网上开放，如果连接通道走默认的 HTTP 协议，将十分不安全。因此有必要给这些服务升级 HTTPS 连接。但这很不容易，基本上要先申请到公开 CA 认证的 HTTPS 证书，然后一个个给这些服务配置上证书和私钥，启用 HTTPS 监听。这十分麻烦并不优雅。

经过一番搜索，发现上述问题其实都有较好的解决方案：证书申请采用 Let's Encrypt 的统配证书，而 HTTPS 通道统一走 Nginx 反向代理。详细方法记录如下。

### 申请 Let's Encrypt 通配证书

2018 年 3 月 Let’s Encrypt 终于宣布支持 ACME v2 and Wildcard Certificate，即通配符证书。非常振奋人心，Good Job！申请通配符证书需要在域名的 DNS 上配置 `TXT` 记录，我们先尝试手动模式申请通配符证书，然后运行一个对应于域名 DNS 服务商的 Certbot 容器来自动申请

- 手动申请

安装 Certbot

```sh
sudo apt install certbot
```

接着运行如下命令给你的域名申请统配符证书（将命令中的域名替换成你自己的）

```sh
certbot certonly --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory --manual-public-ip-logging-ok -d '*.linhongbo.com' -d linhongbo.com
```

Cetbot 会先提示提供邮箱，这里如实填写，以接收证书过期提醒邮件；然后要求配置一个指定值的名为 `_acme-challenge` 的 DNS TXT 记录，我们将其配置到自己的 DNS 服务商上即可；最后稍等片刻，先用 [dns.google.com](https://dns.google.com) 查询一下 TXT 记录是否生效，确认生效后回车。待 Let's Encrypt 校验域名所有权后，会签发证书，成功申请的输出如下：

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/linhongbo.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/linhongbo.com/privkey.pem
   Your cert will expire on 2018-06-12. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:
 
 
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Cetbot 将证书和私钥归档在 `/etc/letsencrypt/archive/yourdomain.domain/` 下，但我们应通过其额外提供的软连接访问，即

```
/etc/letsencrypt/live/linhongbo.com/fullchain.pem
/etc/letsencrypt/live/linhongbo.com/privkey.pem
```

- 自动申请

自动申请能够自动化配置 DNS TXT 记录，这需要 Certbot 安装对应域名服务商的插件，处于方便考虑，直接采用 Docker 容器运行。

首先，创建一个用于访问 DNS 服务商 API 的接口配置文件，内容根据你实际 DNS 服务商提的而不同，我的例子是 [Cloudflare](https://dash.cloudflare.com/profile/api-tokens)

```sh
vim cloudflare.ini

# Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234
```

接着在 Docker Hub 上查找到对应的 [Certbot 镜像](https://hub.docker.com/u/certbot/)，我的例子是 certbot/dns-cloudflare，然后运行如下命令申请证书

```sh
sudo docker run -it --rm --name certbot \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            -v "/path/to/cloudflare.ini:/cloudflare.ini" \
            certbot/dns-cloudflare certonly --preferred-challenges dns --dns-cloudflare --dns-cloudflare-credentials /cloudflare.ini  -d *.linhongbo.com -d linhongbo.com --server https://acme-v02.api.letsencrypt.org/directory
```

### 配置 Nginx 反向代理

openmediavault 预置安装了 Nginx（被其管理界面使用），我们直接复用这个 Nginx 作为反向代理服务器。

以配置 Nextcloud 为例，首先创建 Virtul Host 配置文件：

```sh
vim /etc/nginx/sites-available/nextcloud

server {
    #常规 HTTP/HTTPS 端口，用于局域网访问。遵循一端口全局仅配置一次 ipv6=off 规则，例如 openmediavault 的 80 或 443 端口的配置语句已经包含了 `ipv6only=off`，下面就要去除掉对应的 `ipv6only=off`
    listen [::]:80 ipv6only=off;
    listen [::]:443 ipv6only=off ssl http2;
    
    #出于安全考虑，如果该服务需要被公网访问，额外监听一个端口作为转发专用端口，以对接路由器 WAN 侧
    #listen [::]:8443 ipv6only=off ssl http2;

    server_name nextcloud.linhongbo.com;
    
    ssl_certificate /etc/letsencrypt/live/linhongbo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/linhongbo.com/privkey.pem;
    ssl_verify_client off;
    proxy_ssl_verify off; 
    location / {
        proxy_pass  http://localhost:9096;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

其中，

- `ipv6only=off` 表示监听的 ipv6 socket 既可以处理 ipv6 也可以处理 ipv4 数据包（这是 Linux 的一个新特性，新版 Ningx 默认为 `on`，即在该端口上仅监听 ipv6 数据包）。这个选项可以使配置更简洁，而无需（且不能）额外配置 ipv4 监听语句，形如 `listen 0.0.0.0:80`。但要注意如果有多个 `server` 块监听同一端口，这种写法要求该端口的 `ipv6only=off` 选项**必须**且**只能**出现一次，否则会导致 `Address already in use` 错误（两个端口同时处理 ipv4 请求）或无法处理 ipv4 请求。也就是说由于 openmediavault WEB 界面自身监听的 80 端口和 443 端口（如果启用 HTTPS）在其配置文件中已经配置了 `ipv6only=off`，遵循只能出现一次的规则，你额外的 Virtual Host 配置中 80 和 443 端口就不需且不能再配置 `ipv6only=off` 了。
- `server_name` 指定服务对应的域名
- `ssl_certificate` 和 `ssl_certificate_key` 分别填写先前申请的 Let's Encrypt 证书、私钥文件；
- `proxy_ssl_verify off` 表示关闭 Nginx 到上游服务器（Nextcloud 容器服务）的 SSL/TLS 校验，由于我是反向代理到 Nextcloud 容器的 HTTPS 服务，因此该选项必须。
- `proxy_set_header Host $http_host` 表示反向代理时替换掉 HTTP Header 中的 host 参数，这对 Nextcloud 是必要的，否则会报域名不被信任的错误；
- `proxy_pass https://localhost:9443` 设置被代理容器监听的本地回环地址和端口号；
- `proxy_set_header X-Real-IP $remote_addr` 和 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` 指示 Nginx 发起反向代理请求时增加原始 IP 字段到 HTTP Header，这对一些依赖源 IP 来认证客户端的 Web 服务非常重要，例如 Emby，由于反向代理时 Nginx 位于 localhost 或局域网，如不设置此参数 Emby 将认为请求是来源于局域网，从而造成安全功能误判。因此建议所有被代理服务配置上此选项。
- `http2` 支持 http2，加速网站加载。注意仅 HTTPS 支持此特性。

将上述配置文件拷贝到 `/etc/nginx/sites-enabled` 目录使能起来，这里应用软链接

```sh
ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
```

命令 Nginx 重新加载配置

```sh
nginx -s reload
```

## 端口转发（公网访问）

最后，为了能在公网访问 openmediavault 上的 WEB 服务，需要在路由器的防火墙规则中增加一条 WAN:8443 -> openmediavault:8443 的 tcp 端口转发规则，这里 WAN 域端口选择 `8443` 而非 HTTPS 默认的 443 端口是因为该端口已运营商防火墙封锁了。此外由于 80 端口亦被封锁，博主不再配置 HTTP（80）端口的转发规则，因为不会带来任何访问上的裨益（网址仍要指定端口才能访问）。

## VPN

VPN 可以在公网和家庭局域网之间创建隧道，从而容易访问内网的各项服务。很多路由器都提供了这项功能。如果你想使用 Shadowsocks 来替代 VPN，可以参考博主的这篇文章：[OpenWrt 安装 Shadowsocks Server](https://linhongbo.com/posts/shadowsocks-server-on-openwrt/)

## 常见问题

### 反向代理性能差

默认配置，Nginx 会将后端请求缓存起来（如果缓存空间不够还会写入本地文件）再发送给客户端，这样就会对 Emby 等大流量数据流场景形成性能瓶颈，导致卡顿现象。

解决办法是在对应域名的 `server` 块中加入如下一行配置（建议在 `nginx.conf` - `http` 中全局配置）

```
proxy_buffering off;
```

这样，Nginx 获取到后端响应就会立即传送给客户端。
