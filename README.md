# GoogleVideo QUIC Node Scanner

一个自动化工具，用于扫描和筛选 `googlevideo.com` 全部子域名中可直连的 QUIC 视频节点 IP。

---

## 背景知识

> 以下内容来自 [聊聊那些翻山越岭的神奇操作（下）-前篇](https://blog.kenxu.top/post/bypass-gfw-p3/)，作者 Ken。

### 回顾

经过前两篇文章的介绍，各位应该了解到，当下若想绕过防火墙对网站的封锁，基本需要满足以下三点：

1. 需要能获取到网站对应的正确 IP
2. 网站对应的 IP 不能被封锁
3. 需要绕过防火墙的 SNI 阻断

其中 1 可以通过境外的 DoT 或 DoH 服务解决。若网站使用 AnyCast 技术，则可以从已公布的 IP 范围内扫描并筛选出可用的 IP 以解决 2。对于 3，有三种绕过方法：**域前置**，**QUIC** 和 **ECH**。

将目光转向 QUIC 和 ECH，虽然作为新兴的技术尚未得到普及，但我们依然能利用这两项技术做一些有趣的事情。

### HTTPS RR

在进行以下实践之前，有必要详细了解一下这个比较新的 DNS 记录类型：HTTPS

从 HTTP/0.9 到 HTTP/3，HTTP 协议在速度，效率，甚至底层传输协议上都有了翻天覆地的变化。随着 HTTP 协议的不断发展，一个问题浮现出来：客户端和服务端要怎样得知对方支持哪些协议？目前广泛流行的协商方案有：TLS ALPN 扩展（[RFC 7301](https://datatracker.ietf.org/doc/html/rfc7301)）和 HTTP Header 中的 `alt-svc` 字段（[RFC 7838](https://datatracker.ietf.org/doc/html/rfc7838)）。前者通过 TLS Client Hello 握手包中的扩展告知服务端客户端可用的协议，后者则通过 HTTP 头告知客户端服务端可用的协议。

这一套协商流程有什么问题？由于 HTTP/2 及之前的协议均基于 TCP，因此这套协商流程工作十分良好。但 HTTP/3 协议基于 UDP，因此无法在使用 TCP 连接的时候直接升级到 HTTP/3，只能新建一条连接。因此早期，客户端若想使用 HTTP/3 与服务端连接，必须先使用 HTTP/2 或之前的协议连接至服务端，待识别到服务端发来的 HTTP `alt-svc` 响应头包含 `h3` 的时候，才能将后续的连接升级至 HTTP/3。

很明显，这样做的效率很低，那要如何才能第一时间让客户端与服务端通过双方支持的最新协议建立连接呢？突破点就在一切连接建立之前的 DNS 查询流程中。只要将服务端支持的协议放入 DNS 记录中，客户端就能通过查询 DNS 得知服务端所支持的协议了。HTTPS 这一记录类型就提供了这一功能（[RFC 9460](https://datatracker.ietf.org/doc/html/rfc9460)）：

> The SVCB ("Service Binding") and HTTPS resource records (RRs) provide clients with complete instructions for access to a service. This information enables improved performance and privacy by avoiding transient connections to a suboptimal default server, negotiating a preferred protocol, and providing relevant public keys.

为了达成以上所述的效果，HTTPS 记录类型包含了以下几个关键字段：

- `alpn`：服务端所支持的协议列表
- `ipv4hint` 和 `ipv6hint`：包含服务端所使用的 IP 地址，客户端可直接使用这些地址与服务端建立连接，而不必等待常规的 A/AAAA 查询返回结果
- `ech`：一个 `ECHConfigList` 类型的数据，提供 ECH 所需的公钥和服务端域名等信息

不难发现，通过查询域名对应的 HTTPS 记录，客户端即可得知服务端支持的协议与是否支持 ECH，无需通过上文提到的协商流程即可直接与服务端发起 HTTP/3 和/或 ECH 连接，减少额外的协议探测与连接切换开销。

### QUIC

中篇中提到，虽然 QUIC 在握手阶段不提供机密性，但防火墙在根据 QUIC 连接握手阶段的 SNI 信息阻断连接方面的能力较弱。Chrome 和 Firefox 均在 QUIC 握手阶段启用了 **SNI 分片功能**，SNI 会被分散到多个 UDP 包中，而 **GFW 目前不具备重组这些包的能力**。也就是说，只要我们能与目标网站建立 QUIC 连接，就能绕过防火墙的阻断。为了做到这一点，必须要满足以下两个要求：

1. 网站本身必须支持 QUIC
2. 浏览器需要知道网站支持 QUIC，并在首次连接就使用 QUIC

对于第一点，虽说 QUIC 作为较新的协议仍未得到广泛支持，但好在部分国外 CDN 厂商已经对 QUIC 提供了支持。那要怎样才能得知网站的支持情况呢？我们可以通过观察网站的 HTTPS 记录和/或 HTTP 响应头中的 `alt-svc` 字段以确认：

![HTTPS RR](https://blog.kenxu.top/post/bypass-gfw-p3/1.png)

![alt-svc](https://blog.kenxu.top/post/bypass-gfw-p3/2.png)

虽说以上两点可以覆盖大多数网站，但依然存在部分支持 QUIC，但并未通过任何方式告知客户端的网站。对于这种情况，我们可以使用 cURL 进行测试，以 `reddit.com` 为例，使用以下指令强制使用 QUIC 与其建立连接：

```bash
curl https://www.reddit.com --http3-only -I
```

![cURL 测试](https://blog.kenxu.top/post/bypass-gfw-p3/3.png)

测试发现返回 200 状态码，证明网站确实支持 QUIC。

理想状态下，网站应为其域名添加 HTTPS 类型的解析记录并在 `alpn` 中包含 `h3`，否则仍然需要先建立 HTTP/2 连接再通过 `alt-svc` 升级到 HTTP/3。但这种常规的升级对被阻断的网站来说是致命的，毕竟在 TLS 握手阶段防火墙就会精准阻断连接，根本等不到升级那一步。

这种情况下，为了让浏览器直接通过 QUIC 与目标网站建立连接，我们需要自行搭建一个 DNS 转发器，手动为对应网站添加 HTTPS 类型记录，并将浏览器使用的 DNS 服务器指向我们自建的转发器。

### Google

首先我们需要简单了解 Google 系网站服务器的组成。总体来说，Google 大致有以下几类服务器：

1. **GWS** - Google Web Service，提供 Google 系网站服务
2. **GGC** - Google Global Cache，由 ISP 提供，旨在缓存 Google 系网站的资源以加速用户访问
3. **GVS** - Google Video Server，提供 YouTube 上的视频资源分发服务
4. **GCC** - Google China Cache，阉割版的 GGC，向中国大陆用户提供有限度的服务

1 和 2 的 IP 地址可以混用，也就是说，只要找到任意一个未被防火墙封禁的 GWS 或 GGC IP，即可将其套用到所有 Google 系网站上。而 3 则具有严格的对应关系，YouTube 上不同的视频资源不可套用同一个 IP。

作为 QUIC 协议的主要开发者之一，Google 系网站对 HTTP/3 的支持情况如何呢？对于各类服务的域名，均能在 HTTP `alt-svc` 头中找到 `h3` 的身影，但 Google 只为其主域名 `www.google.com` 添加了 HTTPS 记录，其余域名均无相关记录。

![h2 甚至在 h3 之前，意味着浏览器会优先使用 HTTP/2 建立连接](https://blog.kenxu.top/post/bypass-gfw-p3/8.png)

GVS 的情况则截然相反，可以看得出来 Google 十分想让你在观看 YouTube 视频的时候使用 QUIC：

![甚至只有 h3](https://blog.kenxu.top/post/bypass-gfw-p3/9.png)

很明显，我们需要为 Google 系的网站添加 HTTPS 记录。不过在这之前还有一个问题需要解决：GWS 的 IP 基本已经被防火墙封锁殆尽，如何找到可用的 IP 地址呢？

天无绝人之路，IPv4 被封了，我们还有 IPv6！查询 `www.google.com` 的 AAAA 记录可得一系列 IPv6 地址，且 `/64` 这一整块地址均可用于访问 GWS。令人惊喜的是，防火墙对于 IPv6 依然采取封禁单个 IP 的策略，也就是说，我们只需要从这个 /64 段中任意挑选一个未被封锁的 IP 加入 DNS 记录中即可。

同样，对于 GVS，虽然 IPv4 地址的情况与 GWS 相同，但 IPv6 大多幸免于难，加上已经存在的 HTTPS 记录，我们甚至不需要做什么就能正常加载 YouTube 上的视频资源。

> 原文链接：https://blog.kenxu.top/post/bypass-gfw-p3/

---

## 本工具的作用

本工具正是基于以上原理，针对 GVS（`*.googlevideo.com`）进行大规模扫描：

1. 从证书透明度日志获取 `googlevideo.com` 的全部历史子域名
2. 通过 DoH 解析每个子域名的真实 IP
3. 使用原生 ICMP Ping 测试哪些 IP 在你的网络环境下可直连
4. 筛选出可用节点，配合自建 DoH 代理返回 `alpn=h3` 的 HTTPS 记录，强制浏览器走 QUIC

## 特性

- 🚀 **高并发**：DNS 解析 20 线程 + Ping 测试 50 线程并行
- 📊 **实时进度**：每个操作都有 `[当前/总数]` 进度显示
- 💾 **断点续传**：每轮结束立即保存文件，中断后重启自动从存档恢复
- 🔄 **智能重试**：失败域名重新查 DNS 获取新 IP，而非傻 Ping 旧 IP
- ⏹️ **随时退出**：`Ctrl+C` 或直接关闭窗口，已完成轮次的数据不会丢失
- 🛡️ **绕过代理**：Ping 使用系统原生 `ping.exe`，不受 Clash/V2Ray 等代理软件干扰
- 🔧 **零依赖**：纯 Python 标准库，无需 `pip install` 任何第三方包

## 使用方法

### 环境要求

- Windows 10/11
- Python 3.8+

### 快速开始

```bash
python googlevideo_scanner.py
```

### 自定义 DoH 服务器

编辑脚本顶部的配置区：

```python
# 默认使用 Google Public DNS
DOH_URL = "https://dns.google/dns-query"

# 也可以替换为其他 DoH 服务器，例如：
# DOH_URL = "https://cloudflare-dns.com/dns-query"
# DOH_URL = "https://your-own-doh-server.com/dns-query"
```

## 输出文件

| 文件名 | 说明 |
|--------|------|
| `googlevideo_resolved.txt` | 第一轮 DNS 解析出的全部 `域名 IP` 记录 |
| `googlevideo_success.txt` | Ping 测试通过的直连可用节点 |
| `googlevideo_fail.txt` | Ping 测试失败的记录（后续轮次会重试） |

文件格式均为每行一条 `域名 IP` 记录，例如：

```
rr5---sn-axq7sn76.googlevideo.com 142.250.5.161
rr3---sn-npoe7ney.googlevideo.com 173.194.182.27
```

## 工作流程

```
第 1 轮:
  ┌─ 从 crt.name 获取全部子域名
  ├─ 通过 DoH 解析全部子域名的 IP
  ├─ 保存解析结果 → googlevideo_resolved.txt
  ├─ 对去重后的 IP 进行 Ping 测试
  ├─ 通过 → googlevideo_success.txt
  └─ 失败 → googlevideo_fail.txt

第 2 轮:
  ┌─ 读取 fail.txt 中的域名
  ├─ 重新查询 DNS（可能获得新 IP）
  ├─ 对新 IP 进行 Ping 测试
  ├─ 通过 → 追加到 success.txt，从 fail.txt 删除
  └─ 失败 → 留在 fail.txt

第 3 轮、第 4 轮……重复以上流程
直到 fail.txt 清空或用户手动停止
```

## 常见问题

### 为什么有大量域名显示"无记录"？

`crt.name` 返回的是 Google 历史上申请过证书的所有子域名（可能跨越数年）。Google 会频繁新建和销毁边缘节点，所以大量旧域名已被 Google 废弃（DNS 返回 NXDOMAIN）。这是正常现象。

### 为什么 Ping 通的 IP 比例很低？

这取决于你的网络环境。在某些地区，大量 Google IP 段可能被防火墙封锁。扫描出的可直连 IP 正是在你特定网络环境下"漏网"的优质节点。

### 如何重新开始全量扫描？

删除当前目录下的 `googlevideo_fail.txt` 和 `googlevideo_success.txt`，然后重新运行脚本即可。

## License

MIT

## 服务端部署 (DoH 代理)

除了在本地运行扫描工具外，你还需要在**境外 VPS** 上部署配套的 DoH 代理服务器。这个 DoH 服务器负责：
1. 拦截指定域名并注入 \lpn=h3\ 记录强制 QUIC。
2. 命中我们扫描出来的优质节点时，直接返回可直连的 IP。

### 部署步骤

在你的境外 VPS 上执行以下命令下载并一键安装：

\\ash
# 1. 下载服务端压缩包并解压 (请将 URL 替换为你的 GitHub Releases 地址)
wget https://github.com/your-username/your-repo/releases/download/v1.0/doh-server.zip
unzip doh-server.zip -d doh-server
cd doh-server

# 2. (可选) 上传本地扫描结果
# 如果你在本地运行 scanner 扫出了结果，请将 googlevideo_success.txt 上传到当前目录
# 安装脚本会自动将其转换为 JSON 映射表并加载

# 3. 运行一键安装脚本
chmod +x install.sh
sudo ./install.sh
\
安装完成后，DoH 服务将会在 VPS 的本地 W.0.0.1:3053\ 监听。

### 反向代理配置

由于 DoH 必须使用 HTTPS，你需要使用 Caddy 或 Nginx 进行反向代理并配置 SSL 证书。

**Caddy 配置示例：**
\\caddyfile
your-doh-domain.com {
    handle /doh/* {
        reverse_proxy localhost:3053
    }
}
\
配置好 HTTPS 反代后，将本地扫描工具中的 \DOH_URL\ 修改为你自己的地址即可。
