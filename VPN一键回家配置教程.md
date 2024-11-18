# VPN一键回家配置教程 v1.0

## 目录

[[_TOC_]]

## 概述

这里主要介绍一下在外地访问家中网络的方法。通过完成下面介绍的配置，能够通过VPN接入家里的局域网，从而达到远程访问家中设备的目的。

什么你问为什么不直接暴露家里设备的端口到公网？因为不敢，反正我是不敢的，这样太危险了。但是如果我们只暴露一个VPN的端口，这就好多了。

为了达成这样的功能，我们需要满足以下条件:

1. 获取家里当前的公网IP
2. 在家中架设一个VPN服务器
3. 穿过路由器的防火墙和NAT来访问家里的VPN服务器
4. 在当前使用的设备上，IP包能被正确路由
5. 在家里网络设备上，IP包能被正确路由

对于#1，IPv6+DDNS能够很好的完成工作。

对于#2，有些软路由或者硬路由是自带VPN功能的，或者可以直接在软路由上安装VPN软件，总之你需要一台VPN服务器。

对于#3，如果你是用的路由器自带的VPN，那一般不需要特殊设置。否则就需要配置端口转发或者端口映射，从而能够从公网连接到VPN服务器。

### 涉及到的知识点：

1. 网络基础（IPv4，IPv6，路由表等）
2. 配置DDNS
3. Linux基本操作，配置
4. OpenVPN的配置

## 获取公网IP

IPV4的公网地址，基本上是没有了。
不过，按照目前（2024年11月）的政策，上头正在大力推行IPv6和去NAT化，所以目前**所有直接接入运营商的设备都是能够获取公网的动态IPv6的**。
也就是说你的手机（当使用数据流量时）和路由器（直接PPPoE拨号）都是有公网动态IPv6的。
所以无论你身处何方，只要有网，至少可以通过手机热点的方式直连回家。

为了获取公网IPv6，首先要确保路由器直接接入了运营商网络。
所以先要确保路由器使用了**PPPoE拨号**而不是光猫拨号。如果发现是使用了光猫拨号，请联系运营商改成路由器拨号。
（具体步骤取决于当地运营商的政策，所以就不在这里介绍具体步骤了。

确认路由器使用PPPoE拨号后，进入路由器管理界面，找到IPv6的开关，打开之后路由器应该就能获得一个公网IPv6地址了。
接下来可以进一步检查这个地址是否能被实际访问到。此时可以让电脑连接手机热点再ping一下这个地址来确认能从公网与路由器建立通讯。

关于IPv6，这里还需要讲一下确保家里局域网安全性的设置。
因为IPv6的地址空间非常大，所以在路由器打开IPv6后可能会为家里的所有设备都分配公网IPv6地址，这大大增加了这些设备被攻击和入侵的机会。
所以支持IPv6的路由器一般都会提供防火墙和NAT6的功能，选择其一即可保护家里的设备。
鉴于NAT6会比较好获取路由器的公网IPv6，本文会基于打开NAT6的选择来讲解后续配置。

## 配置DDNS服务

经过上面的步骤，虽然我们已经确保当前能够从公网与路由器建立通讯，但是路由器获取到的IP是一个动态IP。也就是说运营商每过一段时间都会给路由器换一个IP，当出门了要连接的时候就肯定不是这个IP了。这个状况无法改变，除非你愿意掏天价问运营商买企业宽带。所以我们需要掏出**DDNS**！ 

所谓DDNS，其实就是普通的DNS，只不过会有个定时检测目标IP地址然后去更新DNS记录的操作。我们只需要每隔1分钟就检测一下当前路由器的公网IP，如果发生了改变，就去更新DNS记录。虽然可能在IP变动后最多有1分钟的延迟，但是鉴于一般IP要过几天才会重新分配一次，这不会导致太多问题。

要配置DDNS，首先需要一个域名。如果你还没有域名，那么可以找个便宜的地方买个域名，国内国外都可以。建议国外买，比较方便。

那么现在可以分配一个听起来不错的子域名来作为家庭路由器的域名。假如我已经有了一个叫`awesome.domain`的域名，那么我们可以将`home.awesome.domain`作为家里的域名。下面的内容都会以这个域名为例，记得实际配置时换成自己的域名。

### 选择现成的DDNS解决方案

市面上有非常多的DDNS提供商，例如花生壳啥的。有些提供商甚至还提供免费的域名，就很方便。可以看看。

### 手搓一个DDNS解决方案

但是，鉴于以上DDNS解决方案不一定适合我们的需求（没法给路由器随便安装软件，没法支持ipv6），以及DDNS实在是过于简单有手就能搓，本着一毛不拔的白嫖精神，我们其实可以把域名托管给Cloudflare，然后手搓一个通过调用Cloudflare API来更新记录的DDNS方案。

如果你的路由器是个可以随便折腾的软路由，那非常建议把DDNS跑在这个上面。这样的话可以直接获取本机的IP地址，那么把IP地址检测频率提高到每秒一次，岂不美哉？如果你的路由器不能随便折腾，那么也有以下选择：

| 部署方案                        | 优点                                                         | 缺点                                                         |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 跑在路由器的docker里            | 不额外花钱买设备，不额外耗电跑设备，不占地方，绿色环保，上佳之选 | 不是所有路由器都能跑docker<br />就算能跑docker，也是半残废，折腾起来太麻烦了 |
| 跑在家里PC上                    | 不额外花钱买设备，暂时保住了钱包                             | PC得一直开机，长期来看还是没保住钱包                         |
| 跑在家里PC上<br />+远程开机方案 | 只花一点点钱，算是保住了所有钱包                             | 并不是所有电脑都有地方装远程开机卡的                         |
| 再买个开发板/软路由             | 节能环保，而且也不怎么占地方                                 | 说好的不氪金呢？                                             |

这边我们以部署在开发板上为例，讲一下怎么搓DDNS。

一般来讲，这种简单的DDNS方案，只需要一个脚本，然后在路由器上定时运行这个脚本就可以了。这个脚本的功能就是检测当前的公网IP，然后调用Cloudflare的API来更新DNS记录。

Linux上最常用的脚本语言是bash，所以我们可以写一个bash脚本来实现这个功能。但是根据JS定律，一切可以用JS实现的东西最终都会用JS实现，所以我们这里用typescript来写这个脚本。

```typescript
import { promises as dns } from "dns";
import axios from "axios";
import Cloudflare from "cloudflare";

const zone_id = "############## replace with your zone id ############";
const apiToken = "############## replace with your api token ############";
const domainName = "home.awesome.domain";  // replace with your domain name

function log(...args: any[]) {
  console.log(new Date().toISOString(), ...args);
}

async function getPublicIpV6() {
  const response = await axios.get("https://ipv6.icanhazip.com");
  if (response.status !== 200) {
    throw new Error(
      "Failed to get public ip v6, status code: " + response.status
    );
  }
  return (response.data as string).replace(/\s+/g, "");
}

async function getCurrentDNS(cf: Cloudflare) {
  const records = await cf.dns.records.list({
    zone_id,
  });

  const targetRecord = records.result.find((r) => r.name === domainName);
  if (!targetRecord) {
    throw new Error("Cannot find target record");
  }
  if (!targetRecord.id) {
    log("Record: ", targetRecord);
    throw new Error("record.id is empty");
  }
  return [targetRecord.id, targetRecord.content as string];
}

async function checkAndUpdate() {
  const cf = new Cloudflare({
    apiToken,
  });

  const [[recordId, currentAddr], trueAddr] = await Promise.all([
    getCurrentDNS(cf),
    getPublicIpV6(),
  ]);
  log("Current DNS: ", currentAddr);
  log("True IP: ", trueAddr);
  if (currentAddr === trueAddr) {
    log("No need to update DNS");
    return;
  }

  log("Updating DNS...");
  cf.dns.records.edit(recordId, {
    type: "AAAA",
    zone_id,
    name: domainName,
    content: trueAddr,
  });
  log("DNS updated successfully");
}

async function sleep(ms: number) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

async function main() {
  do {
    try {
      log();
      log("=== Start working ===");
      await checkAndUpdate();
    } catch (e) {
      log("Error: ", e);
    }
    await sleep(1000 * 60); // do it every 1 minute
  } while (true);
}

main();

```
可以看到，这个脚本还需要填写一些参数，例如`zone_id`和`apiToken`。这些参数可以在Cloudflare的控制台上找到。
一旦填写好了这些参数，就可以在某个设备上运行这个脚本了。

因为我们开启了IPv6的NAT，所以这个脚本通过`https://ipv6.icanhazip.com`获取的IP地址就是路由器的公网IPv6地址。因此这个脚本就能够实现检测当前公网IP是否发生变化，如果发生变化就更新DNS记录。

由于我没有备份我的整个项目，所以这个脚本是我现写的，可能会有一些错误，也没有package.json。需要自己写一个package.json，然后`npm install`一下。相信这些都难不倒你。

接下来，我们要验证一下这个脚本是否能够正常工作。可以在本地运行这个脚本，然后在Cloudflare的控制台上查看DNS记录是否发生了变化。也可以打开命令行ping一下`home.awesome.domain`，看看是否能够ping通。甚至可以用一些测试网络延迟的网站测试一下

## 配置VPN

首先我们来讲一下什么是VPN。这里的VPN和我们常用的翻墙软件经常被混淆在一起，但是其实它们的有根本性的不同的。
翻墙软件一般都是代理（proxy），需要应用支持才能使用，一般都是基于HTTP的协议在使用。
而VPN则是会在你的电脑上插入一张虚拟的网卡，有虚拟的IP，还配置了路由表，确保该去内网的IP数据包都去内网，而该去其他地方的数据包都不受影响，而APP是不知道用了VPN的。
例如，ping一个IP的时候，代理是不起作用的，而VPN则能工作。

具体到要使用什么VPN协议呢，我这里的推荐还是OpenVPN。理由如下：

1. 与PPTP之类的协议相比，OpenVPN对在IPv6网络上建立虚拟信道支持比较友好
2. Wireguard只能使用UDP协议来建立虚拟信道，而国内网络环境对UDP不太友好，而OpenVPN可以选择使用TCP协议。

和手搓DDNS类似，我们也需要一个地方来部署VPN，那么还是以下选择：

| 部署方案                      | 优点                                                         | 缺点                                                         |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 跑在路由器的docker里          | 不额外花钱买设备，不额外耗电跑设备，不占地方，绿色环保，上佳之选 | 不是所有路由器都能跑docker<br />就算能跑docker，也是半残废，折腾起来太麻烦了 |
| 跑在家里PC上                  | 不额外花钱买设备，暂时保住了钱包                             | PC得一直开机，长期来看还是没保住钱包                         |
| 跑在家里PC上<br />+远程开机卡 | 只花一点点钱，算是保住了所有钱包                             | 并不是所有电脑都有地方装远程开机卡的                         |
| 再买个开发板/软路由           | 不太耗电，而且也不怎么占地方                                 | 说好的不氪金呢？                                             |

`todo: 详细配置教程`

既然我们已经把DDNS部署在了开发板上，那么我们索性就把VPN也部署在开发板上。

首先我们来讲一下家庭内网的网络拓扑，以便更好的理解VPN的配置。
一般来讲，家用内网会有一个路由器作为主路由，也是唯一一个直接连接到运营商网络的设备。
其他设备都是连接到这个路由器上的。
家庭内网的IPv4地址一般是私有地址，我们假设使用的是`10.24.0.0 ~ 10.24.255.255` 这个IP段，其中主路由的内网地址是`10.24.0.1`。
而我们的开发板的IP地址是`10.24.1.1`（我们最好使用路由器的DHCP静态IP分配功能给开发板分配一个好记的固定的地址以方便后续使用）。
然后我们在内网中还有一台设备，假设是一台NAS，它的IP地址是`10.24.2.1`。有了这些参数，我们就知道怎么配置VPN了。

接下来是配置VPN的具体步骤。

### 安装openvpn软件

首先安装openvpn 和easy-rsa软件包。easy-rsa是用来生成证书的。

```bash
sudo apt update

sudo apt install openvpn easy-rsa
```
只要这样就可以安装好openvpn和easy-rsa了。
但是这只是第一步，麻烦的是后面的配置。

### 配置openvpn

首先我们要生成证书。我们可以使用easy-rsa来生成证书。

```bash
# 首先我们可以在用户目录下面创建一个目录，用来存放相关的文件
mkdir ~/openvpn
cd ~/openvpn

# 以下借鉴了 https://cloud.tencent.com/developer/article/2315269 的教程

/etc/openvpn/easy-rsa/easyrsa init-pki #1、初始化，在当前目录创建PKI目录，用于存储整数

/etc/openvpn/easy-rsa/easyrsa build-ca #2、创建根证书，会提示设置密码，用于ca对之后生成的server和client证书签名时使用，其他提示内容直接回车即可

/etc/openvpn/easy-rsa/easyrsa gen-req server nopass #3、创建server端证书和私钥文件，nopass表示不加密私钥文件，提示内容直接回车即可

/etc/openvpn/easy-rsa/easyrsa sign server server  #4、给server端证书签名，提示内容需要输入yes和创建ca根证书时候的密码

/etc/openvpn/easy-rsa/easyrsa gen-dh  #5、创建Diffie-Hellman文件，密钥交换时的Diffie-Hellman算法

/etc/openvpn/easy-rsa/easyrsa gen-req client nopass #6、创建client端的证书和私钥文件，nopass表示不加密私钥文件，提示内容直接回车即可

/etc/openvpn/easy-rsa/easyrsa sign client client  #7、给client端证书签名，提示内容输入yes和创建ca根证书时候的密码
```

这样就在当前目录生成了一堆证书文件。当前的目录结构就是这样的了：
``` bash
.
├── easyrsa                    #管理命令
├── openssl-easyrsa.cnf
├── pki
│   ├── ca.crt                #ca根证书，服务端与客户端都需要用
│   ├── certs_by_serial
│   │   ├── 633C217979C7B5F1D0A9ECA971006F96.pem
│   │   └── 857F9B2E3F6C3D35934672212343B42D.pem
│   ├── dh.pem                #认证算法 服务端
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.attr.old
│   ├── index.txt.old
│   ├── issued
│   │   ├── client.crt        #客户端证书
│   │   └── server.crt        #服务端证书
│   ├── openssl-easyrsa.cnf
│   ├── private
│   │   ├── ca.key
│   │   ├── client.key        #客户端私钥
│   │   └── server.key        #服务端私钥
......
```

接下来，是编写配置文件。我们需要编写两个配置文件，一个是服务端的配置文件，一个是客户端的配置文件。客户端的配置文件我们在下面的配置客户端章节中再讲。这里先讲服务端的配置文件。

首先，我们需要选定服务器使用的端口。为了防止被运营商盯上，我们一般不会使用常见的端口，例如80，443等，而是使用一个比较大的端口，比如`64333`。
同时，我们需要选择使用的通讯协议。虽然很多教程都推荐使用UDP协议，但是由于UDP协议在国内网络环境下不太友好，会被QOS和丢包，所以我们这里选择使用TCP协议。
然后我们需要决定给客户端分配的IP地址。为了不和当前的内网IP地址冲突，我们这里可以选择一个不常用的私有地址段，例如`192.168.117.0 255.255.255.0`。
另外，我们需要让客户端知道内网的IP地址范围是`10.24.0.0 255.255.0.0`。这样，当客户端有一个数据包要发往这个地址范围的时候，就知道要通过VPN发出去。而其他的数据包则不受影响。

那么我们的服务端配置文件就是这样的：
``` bash
port 64333                                    #端口
proto tcp                                    #协议
dev tun                                      #采用路由隧道模式
ca /opt/easy-rsa/pki/ca.crt                  #ca证书的位置
cert /opt/easy-rsa/pki/issued/server.crt     #服务端公钥的位置
key /opt/easy-rsa/pki/private/server.key     #服务端私钥的位置
dh /opt/easy-rsa/pki/dh.pem                  #证书校验算法  
server 192.168.117.0 255.255.255.0                #给客户端分配的地址池
push "route 10.24.0.0 255.255.0.0"        #允许客户端访问的内网网段
ifconfig-pool-persist ipp.txt                #地址池记录文件位置，未来让openvpn客户端固定ip地址使用的
keepalive 10 120                             #存活时间，10秒ping一次，120秒如果未收到响应则视为短线
max-clients 100                              #最多允许100个客户端连接
status openvpn-status.log                    #日志位置，记录openvpn状态
log /var/log/openvpn.log                     #openvpn日志记录位置
verb 3                                       #openvpn版本
client-to-client                             #允许客户端与客户端之间通信
persist-key                                  #通过keepalive检测超时后，重新启动VPN，不重新读取
persist-tun                                  #检测超时后，重新启动VPN，一直保持tun是linkup的，否则网络会先linkdown然后再linkup
duplicate-cn                                 #客户端密钥（证书和私钥）是否可以重复
comp-lzo                                     #启动lzo数据压缩格式
```
可以发现，我们在这个文件中定义的证书文件路径都是在`/opt/easy-rsa/pki` 目录下。
所以我们接下来需要把之前生成的证书文件拷贝到这个目录下。
然后我们也把这个配置文件保存到`/etc/openvpn/server.conf`。

接下来，我们启动服务器，并且把这个服务设置为开机启动。

``` bash
systemctl start openvpn@server #启动服务
systemctl enable openvpn@server #设置开机启动
```

此时，如果运行以下`ip a` ,就可以看到多了一个`tun0`的网卡，这个网卡就是我们的VPN网卡。
```bash
...
...
4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none 
    inet 192.168.117.1 peer 192.168.117.2/32 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::c172:724:7fe6:f754/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
...
```
可以发现，这个网卡的IP地址在我们之前定义的地址池中。
也就是说这个网卡和后面连上来的客户端的虚拟网卡在通一个虚拟网段当中，他们可以互相进行通讯。

此时我们可以确认VPN的服务端已经在运行了。

## 映射公网端口

只要不是太差的路由器，一般都有端口映射的功能。
就是说当路由器的指定的公网端口收到数据包的时候，会把数据包转给指定内网机器的指定端口。
这样就实现了将VPN端口映射到公网的目的。只要去路由器的设置界面点几下，就可以了！

上面说到，我们选择了`64333`这个端口，然后开发板的IP地址是`10.24.1.1`。
这里我们选择路由器的`53222` 端口作为公网端口。
那么，我们只需要配置将路由器的`53222`端口映射到`10.24.1.1·的`64333`端口就可以了。

可恶，我的小米路由器居然不支持IPv6的端口转发！在同时开启端口转发和IPv6之后，如果向路由器的IPv4的外网端口发包，可以观察到数据包被正常转发，但是IPv6的外网端口却不行！经过各种研究，发现了一个办法，那就是破解！

### 通过破解解决小米路由器的IPv6端口转发问题

#### 破解小米路由器
详细操作流程可以参考 [[教程] 小米 AX3000 使用 xmir-patcher 工具开启SSH](https://www.right.com.cn/forum/thread-8348196-1-1.html)

项目地址 [xmir-patcher](https://github.com/openwrt-xiaomi/xmir-patcher)。

#### 打开端口转发

在破解ssh进入路由器的shell之后，我们安装一个叫lucky的工具，这个工具包含了端口转发在内的各种有用功能。

链接：[Lucky, 家用软硬路由公网利器](https://lucky666.cn/)

## 配置客户端
Open VPN 的客户端选择有很多。光是官方就提供了两个客户端，一个是OpenVPN Connect，一个是OpenVPN GUI。
这两个客户端都是开源的，可以在官网上下载到。
另外，由于一些公司的组策略，可能会禁止使用这些客户端，这时候可以使用一些第三方的客户端，例如Viscosity。
Viscosity虽然是收费的，但是网上很容易找到破解版或者注册密钥，所以也是可以使用的。

客户端的配置文件如下：
```bash
client
dev tun
proto tcp
remote home.awesome.domain 53222   # 注意更改为自己实际使用的域名和端口
resolv-retry infinite
nobind
ca ca.crt
cert client.crt
key client.key
verb 3
persist-key
comp-lzo
```
除了配置文件之外，我们还需要把客户端的证书文件一起放到客户端的配置文件夹中。
最终的目录结构应该是这样的：
```powershell
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         11/7/2024  11:24 PM           1204 ca.crt
-a----         11/7/2024  11:25 PM           4493 client.crt
-a----         11/7/2024  11:25 PM           1704 client.key
-a----         11/7/2024  11:26 PM            210 client.ovpn
```

接下来，只要将这个配置文件导入到客户端，应该就可以连上VPN了。

此时如果你在客户端运行`ipconfig`，应该可以看到有一个这样的记录：
```powershell
Unknown adapter client:

   Connection-specific DNS Suffix  . :
   IPv6 Address. . . . . . . . . . . : fd53:7061:726b:4c61:6273:5669:7344:4e55
   Link-local IPv6 Address . . . . . : fe80::a32a:14f0:f56c:feb1%8
   IPv4 Address. . . . . . . . . . . : 192.168.117.6
   Subnet Mask . . . . . . . . . . . : 255.255.255.252
   Default Gateway . . . . . . . . . : 192.168.117.5

```
这就是我们客户端的虚拟网卡。

继续运行`route print`，应该可以看到有一个这样的记录：
```powershell
...
...
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
        10.24.0.0      255.255.0.0    192.168.117.5    192.168.117.6     50
...
...
```
这条记录代表着，当客户端要访问`10.24.0.0 255.255.0.0`的这个网段的时候，数据包会通过我们刚刚看到的虚拟网卡发出去。
而不发往这个网段的数据包则参照路由表中的其他规则发出去。
这就解释了为什么这样的VPN可以实现让客户端访问家庭内网的同时，也能访问外网。

## 配置路由

但是，当你以为一切都结束了的时候，你会发现，虽然你可以连上VPN，但是你却不能访问家里的设备。
这是为什么呢？

原来，虽然我们已经配置了VPN，但是VPN并没有帮我们配置好VPN服务器的网络设置。我们需要再配置一下VPN服务器的网络设置。

首先，我们选择让VPN运行在NAT模式。也就是说，从客户端来的数据包，我们需要将它的源地址改成VPN服务器的地址，然后再发出去。这样，对于家里的设备来说，这些数据包就是从VPN服务器发出来的，那么也就知道回复的时候应该回复给VPN服务器了。

要配置NAT，我们需要在VPN服务器上运行以下命令：
```bash
sudo sysctl -w net.ipv4.ip_forward=1 # 打开ipv4转发

sudo iptables -t nat -A POSTROUTING -s 192.168.117.0/24 -d 10.24.0.0/16 -j MASQUERADE # 配置NAT 功能
```

这些命令配置的功能在重启后可能失效，因此我们可以将这些命令写入`/etc/rc.local` 文件，从而在每次开机的时候都会执行这些命令。
需要注意的是，这个文件是以root权限运行的，所以我们不需要在命令前面使用`sudo`。

经过这样的配置，我们就可以在外地访问家中网络了。

## 总结

通过上面的配置，我们就可以在外地访问家中网络了。这样的话，我们就可以在外面访问家里的设备，例如家里的NAS，家里的台式机等等。这样的话，我们就可以非常方便的访问家里的设备了。

鉴于配置过程比较复杂，所以建议大家配置完了之后将笔记本连接到手机的热点上，然后测试一些是否能够用流量访问家里的设备。如果能够访问，那么就说明配置成功了。

## 后续玩法

在成功配置一键回家之后，我们可以用它来做很多事情，例如远程到家里的台式机，远程访问家里的NAS之类的。这里介绍几个比较高端的玩法。

### 远程唤醒电脑

现在的台式机主板基本上都带有远程唤醒功能，只要在BIOS里开启了，就可以通过网络唤醒电脑。这样的话，我们就可以在外面远程唤醒家里的台式机，然后通过远程桌面连接到台式机，就可以在外面使用家里的台式机了。

远程唤醒电脑的原理是通过网络发送一个特定的数据包（叫做Magic Packet）到目标电脑的MAC地址，目标电脑的网卡收到这个数据包后会唤醒电脑。所以要远程唤醒电脑，首先要知道目标电脑的MAC地址。这个MAC地址可以在目标电脑的网络设置里找到。发送Magic Packet的方式有很多：
1. 通过路由器的管理界面发送， 这个方法最简单，但是不是所有路由器都支持
2. 如果你有个一直在线的Linux服务器，可以在上面安装`wakeonlan`这个包，然后通过`wakeonlan`命令发送Magic Packet

下面是一个通过`wakeonlan`命令远程唤醒电脑的例子：
`todo: 详细操作流程`

### 游戏串流

由于P2P直连的延迟相对比较低（实测下来同苏州内跨县的网络延迟在15ms以下）所以串流效果应该是很好的，这里不推荐使用Steam自己的串流方案，因为Moonlight的串流效果更好。