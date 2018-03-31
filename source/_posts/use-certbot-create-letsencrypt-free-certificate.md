---
title: 使用Certbot给你的网站添加HTTPS，部署Let's Encrypt通配符证书
date: 2018-03-16 10:50:15
author: 肖哥
avatar: /yxiao.jpg
authorLink: https://blog.imyxiao.com
authorAbout: https://blog.imyxiao.com/about
authorDesc: Java Developer
categories: certificate
tags:
  - certificate
  - certbot
  - letsencrypt
  - https
description: 以往要在网站上启用HTTPS，是需要向一些证书颁发机构（CA）购买证书的，而如今很多证书机构都提供了免费证书，比如StartSSL、Let's Encrypt、Symantec等等，在国内的话，可以用过阿里云、腾讯云等去申请Symantec DV SSL证书。
photos: /certificate/use-certbot-create-letsencrypt-free-certificate/encrypt.png
---

![certbot](use-certbot-create-letsencrypt-free-certificate/certbot-logo-1A.svg)

## SSL证书

以往要在网站上启用HTTPS，是需要向一些证书颁发机构（CA）购买证书的，而如今很多证书机构都提供了免费证书，比如StartSSL、Let's Encrypt、Symantec等等，在国内的话，可以用过阿里云、腾讯云等去申请Symantec DV SSL证书。

但是这些免费证书的过期时间大多数只有一年，而且大多只有DV证书(单域名证书）。单域名证书，顾名思义，只保护一个域名，使用起来就很不方便，比如我有两个子域名`www.example.com`,`shop.example.com`，那么我需要申请两个证书，子域名越多，管理起来越麻烦。

## 通配符证书

通配符证书可以保护同一主域名下同一级所有的子域名，不限个数，申请证书时，域名填写为 `*.example.com`

比如某公司旗下有一个域名叫 `example.com`，但因为业务需要，解析了很多子域名，比如有`www.example.com；login.example.com；shop.example.com；bill.example.com`

可能多达几十上百个这样的子域名,但为每一个域名都申请一张证书，将会是很昂贵的费用。所以该公司申请了一张支持`*.example.com`的通配符证书，保护了任何前缀的子域名。

<!-- more --> 
## Let’s Encrypt

[Let's Encrypt](https://letsencrypt.org)是由非营利互联网安全研究组织（ISRG）支持的免费，自动化和开放的证书认证机构。他们搞了一个非常有创意的事情，设计了一个[ACME协议](https://ietf-wg-acme.github.io/acme/)，该协议通常在我们的Web主机上运行,目前该协议的版本是v1。

那什么是ACME协议，传统的 CA 机构是人工受理证书申请、证书更新、证书撤销，完全是手动处理的。而 ACME 协议规范化了证书申请、更新、撤销等流程，只要一个客户端实现了该协议的功能，通过客户端就可以向 Let’s Encrypt 申请证书，也就是说 Let’s Encrypt CA 完全是自动化操作的。

## 申请 Let’s Encrypt 通配符证书

为了实现通配符证书，Let’s Encrypt 对 ACME 协议的实现进行了升级，只有 v2 协议才能支持通配符证书，相关的新闻和技术点见：

- [ACME v2 and Wildcard Certificate Support is Live](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579)
- [ACME v2 Production Environment & Wildcards](https://community.letsencrypt.org/t/acme-v2-production-environment-wildcards/55578)

Let’s Encrypt 建议大多数使用shell访问的人使用 Certbot ACME客户端。它可以自动执行证书颁发和安装，而不会停机,也为不想自动配置的人提供专家模式。它很容易使用，适用于许多操作系统，并且具有出色的文档。访问[Certbot站点](https://certbot.eff.org/)以获取适用于您的操作系统和Web服务器的定制说明。这个网址列举了不同的ACME客户端：[client-options](https://letsencrypt.org/docs/client-options/)

唯一不好的地方在于，Let’s Encrypt 证书的持续时间只有3个月，所以证书快到期了需要我们renew一下。

### Certbot

由于从[Certbot站点](https://certbot.eff.org/)上获取的客户端，有些暂时不支持`v2`协议，所以我推荐用Certbot提供的`certbot-auto`脚本。
```
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
```
或者直接去[github.com](https://github.com/certbot/certbot)下载最新的代码
```
git clone https://github.com/certbot/certbot
cd certbot
chmod a+x certbot-auto
```
获取完整的命令行帮助，你可以输入：
```
./certbot-auto --help all
```

客户在申请 Let’s Encrypt 证书的时候，需要校验域名的所有权，证明操作者有权利为该域名申请证书，目前支持三种验证方式：

- dns-01：给域名添加一个 DNS TXT 记录。
- http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件。
- tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件。

而申请通配符证书，只能使用 dns-01 的方式

### 实践

简单介绍下申请步骤（域名：`imyxiao.com`）

```
./certbot-auto certonly -d *.imyxiao.com --manual --manual-public-ip-logging-ok --preferred-challenges dns-01 --agree-tos --server https://acme-v02.api.letsencrypt.org/directory
```

参数说明：

- certonly，表示安装模式，Certbot 有安装模式和验证模式两种类型的插件。
- --manual 表示手动安装插件，Certbot 有很多插件，不同的插件都可以申请证书，用户可以根据需要自行选择
- -d 为那些主机申请证书，如果是通配符，输入`*.imyxiao.com`（可以替换为你自己的域名）
- --preferred-challenges dns-01，使用 DNS 方式校验域名所有权
- --server，Let's Encrypt ACME v2 版本使用的服务器不同于 v1 版本，需要显示指定。
- --manual-public-ip-logging-ok 自动允许公共IP记录
- --agree-tos 同意ACME服务器的用户协议

如果使用`zsh`，添加`-d *.imyxiao.com`可能会报错，请去除这个参数，交互时手动输入

输入命令后输出：
```
Requesting to rerun ./certbot-auto with root privileges...
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for imyxiao.com

-------------------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.imyxiao.com with the following value:

WE3b-3uNQra1erQ-lHUQQifuVvGWSQFAKjqa7F_NtcU

Before continuing, verify the record is deployed.
-------------------------------------------------------------------------------
Press Enter to Continue
```
要求配置 DNS TXT 记录，从而校验域名所有权，也就是判断证书申请者是否有域名的所有权。

上面输出要求给 `_acme-challenge.imyxiao.com` 配置一条 TXT 记录，在没有确认 TXT 记录生效之前不要回车执行。

添加完TXT记录后,输入下列命令，验证TXT记录是否生效：

```
$ dig txt _acme-challenge.imyxiao.com @8.8.8.8

; <<>> DiG 9.10.3-P4-Debian <<>> txt _acme-challenge.imyxiao.com @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1372
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;_acme-challenge.imyxiao.com.	IN	TXT

;; ANSWER SECTION:
_acme-challenge.imyxiao.com. 119 IN	TXT	"WE3b-3uNQra1erQ-lHUQQifuVvGWSQFAKjqa7F_NtcU"

;; Query time: 164 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Mar 16 12:25:40 CST 2018
;; MSG SIZE  rcvd: 112
```

确认生效后，回车执行，输出如下：

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/imyxiao.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/imyxiao.com/privkey.pem
   Your cert will expire on 2018-06-16. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

证书申请成功，证书和密钥保存在下列目录

```
$ sudo tree /etc/letsencrypt/live/imyxiao.com
/etc/letsencrypt/live/imyxiao.com
├── cert.pem -> ../../archive/imyxiao.com/cert2.pem
├── chain.pem -> ../../archive/imyxiao.com/chain2.pem
├── fullchain.pem -> ../../archive/imyxiao.com/fullchain2.pem
├── privkey.pem -> ../../archive/imyxiao.com/privkey2.pem
└── README

0 directories, 5 files
```
这样证书就Ok了，由于证书持续时间只有三个月，所以需要在快到期的时候重新持行一下命令生成新的证书。或者：

```
./certbot-auto renew
```


引用:

> - [Let's Encrypt 终于支持通配符证书了](https://www.jianshu.com/p/c5c9d071e395)
