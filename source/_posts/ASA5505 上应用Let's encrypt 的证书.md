---
title: ASA5505 uses Let's Encrypt SSL certification
date: 2018-12-07 12:20:53
tags:
- ASA5505
- Cisco
- Let's Encrypt
---

# ASA5505 上应用Let's encrypt 的证书

------

公司有一台Cisco的ASA5505，一直被自己用来进行lab网络的VPN Gateway 接入。但是以前一直是使用ASA自身的自签名证书，并没有申请正式的域名证书。看到Let's encrypt的免费证书真的很好用，就打算把这个ASA的自签名证书变成正式的域名证书。

说起来容易，但是做起来发现不是那么容易。首先ASA5505 是比较老款的机器，并不支持Cisco REST API, 也就没有办法使用 [Cisco ASA plugin for Let's Encrypt client](https://github.com/chrismarget/certbot-asa), 这个非常好用而且简单的插件。

既然没法使用certbot，那就只能采用手动方式来安装证书。搜索了一下，只有这个哥们儿的应用场景和我一样[Implementación de Certificados Gratuitos Let's Encrypt en Cisco ASA para Accesos VPN](http://www.3ops.com/implementacion-de-certificados-gratuitos-lets-encrypt-en-cisco-asa-para-accesos-vpn/)，可他的blog是用西班牙文写的。本想找google翻译一下，忽然发现他引用的命令都是可以直接理解的，于是拿出当年玩日文版游戏瞎猜语句的懒劲儿，开始了我的安装步骤。

## 1.申请域名证书
假设你打算应用在VPN网关上的域名是：asa.yourdomain.com 。那么首先使用一台服务器，安装nginx和letsencrypt certbot。因为国内的IPv4网络环境往往都封闭80和443端口，除非你去做了备案，才能保证可用，所以我选择了IPv6。首先设置web服务器网络接口ip是一个全球可路由的IPv6地址，然后在he.net 上面的FreeDNS 把这个IPv6地址指向 asa.yourdomain.com 。

接着配置安装nginx，默认安装即可，我们只是使用它获取https在目标服务器的80/443端口自动验证。

我的服务器是FreeBSD 11.2,所以命令是：

```shell
pkg install nginx
pkg install py27-certbot
```

如何获取证书，这里就不在赘述，网上有很多的教程，在FreeBSD里面的命令是，然后按照提示操作即可：
```shell
certbot certonly --standalone
```

<!--more-->

总而言之，在你设置好 http服务器，并成功完成certbot验证之后，会在 `/usr/local/etc/letsencrypt/live/asa.yourdomain.com/`
目录下，生成对应的证书文件。

## 2.转换证书格式
在刚刚生成的证书目录中，会有以下4个文件：
> * cert.pem: 服务器证书文件
> * chain.pem: 查找用的链式证书
> * fullchain.pem: cert.pem + chain.pem 的合体.  
> * privkey.pem: 证书的私钥


我们需要把证书转换成 PKCS12格式，因为ASA当中使用的证书格式是这个。因此需要用到openssl的命令：
```shell
openssl pkcs12 -export -inkey privkey.pem -in cert.pem -name "asa.yourdomain.com" -out cert.p12
```

这里生成的cert.p12 文件，就是我们要用来在ASA上导入的证书。过程当中要自己新建一个密码口令，后面导入ASA时候会用到。不过这里说实话有个奇怪的地方，在我参考的西班牙语blog当中，那个哥们的命令是：
```shell
openssl pkcs12 -export -inkey privkey.pem -in cert.pem -name "vpn.tuempresa.com.ar" -out cert.p12

```
这个-name后面跟随的域名和他之前写的域名不同，他在原始域名最后加入了".ar"后缀。我查了openssl命令，这里应该没有特殊的含义，所以我就当原文当中的".ar" 是一个笔误。

> -**name** section:
           Specifies the configuration file section to use (overrides default_ca in the ca section).


## 3.ASA上的操作
启动ASDM 的图形化界面，这样方便操作一些。


### 3.1 创建CA
顺序是：
Configuration-Device Management-Ceritifcate Management-CA certificate-Add
在弹出的对话框当中，安装CA证书。
首先随便起个名字: CA-LETSENCRYPT
然后选择  "Paste Certificate in PEM format: "
上传 chain. PEM
最后选择Install certificate.

### 3.2 上传pkcs12格式的证书
顺序是：
Configuration - Device Management - Ceritifcate Management - Identity Certificate - Add
在弹出的对话框当中，导入我们刚刚生成的证书。
首先还是起个名字: SRV-LETSENCRYPT
然后选择  "Import the identity certificate from a file (PKCS12 format with certificate (s) + private Key) "
输入之前openssl转换证书时候创建的密码口令 "Decryption Passphrase: " .
上传 cert.p12 文件
最后选择 Add Certificate

### 3.3 关联VPN的对应配置文件
这里可以重新创建vpn 配置文件，也可以在Configuration - Device Management 里面修改已经存在的配置文件，此处不在赘述具体步骤。


---

以上就是ASA5505使用Let's encrypt 证书的方法，不过记得每隔3个月要更新一下证书，也就是再把步骤1、2、3重来一遍。

