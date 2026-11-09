---
title: 用 curl 和 nmap 做一次 HTTP 与端口排查
date: 2026-11-09 20:10:00
updated: 2026-11-09 21:30:00
slug: curl-http-practice
tags:
  - curl
  - nmap
  - HTTP
  - 端口扫描
categories:
  - 网络实验
---

这次的作业要求是两条：一是任意使用 `curl`，截图记录后发布成博客；二是把过程整理清楚。我顺手把最近一直在用的 `nmap` 也加了进来，因为在实际排查里这两个工具本来就是一对：`nmap` 负责告诉你目标上哪扇门开着，`curl` 负责走到门口往里看一眼。

这篇文章里的每一条命令都是我在 Windows 上真实跑过的，截图是真实输出。前面三节是我原来那篇笔记的内容，后面是这次补全的完整流程。

<!-- more -->

## 先交代环境

| 项目 | 值 |
| --- | --- |
| 系统 | Windows 11（也有 Kali 虚拟机参与） |
| 终端 | Windows PowerShell 5.1 / Windows CMD / Kali 终端 |
| curl | 8.22.0（libcurl/8.22.0，Schannel 后端） |
| nmap | 7.80（Npcap 0.9982） |
| 练习目标 | `https://www.baidu.com`、`https://api.github.com`、`https://httpbin.org`、`scanme.nmap.org` |
| 本地实验目录 | `C:\lab` |

> 目标选择上我只用了三类：自己的机器和家庭内网、公开的 API 测试服务、以及 Nmap 官方明确允许扫描的 `scanme.nmap.org`。扫描别人家的主机不是"技术活"，是另一回事，后面第 25 节会单独说。

---

# 第一部分：curl

## 1. curl 到底是什么

`curl`（Client URL）是一款功能强大的命令行工具，用于在客户端与服务器之间传输数据。它支持 HTTP、HTTPS、FTP、SFTP 等 20 多种协议，是开发者进行 API 调试、网络问题排查和自动化脚本编写的核心工具。

我的理解更朴素一点：**浏览器把"发请求"这件事藏在图形界面后面，curl 把它摊开成一行可以复制、可以存进笔记、可以塞进脚本的命令。**

- 浏览器：地址栏 → 渲染页面，你只能看到结果。
- curl：方法 + URL + 请求头 + 请求体，四样东西全部由你显式给出，服务器返回的原始字节也原样交给你。

所以 curl 真正的用途不是"下载文件"，而是**把一次 HTTP 交互变成可观察、可重复的实验**。

## 2. 先踩一个坑：PowerShell 里的 `curl` 不是 curl

这是 Windows 上最容易翻车的地方，我把它放在最前面。

```powershell
Get-Command curl | Format-Table CommandType, Name, Definition -AutoSize
```

![PowerShell 中 curl 是 Invoke-WebRequest 的别名，curl.exe 才是真正的 curl](/images/curl-http-practice/ps-pitfall.png)

输出是：

```
CommandType Name Definition
----------- ---- ----------
      Alias curl Invoke-WebRequest
```

也就是说，在 Windows PowerShell 5.1 里直接敲 `curl`，执行的根本不是 curl，而是 PowerShell 给 `Invoke-WebRequest` 起的一个别名。它长得像 curl，但参数体系完全不一样：

| 你以为在写 | 实际含义 |
| --- | --- |
| `curl -I https://a.com` | PowerShell 会报"找不到参数 I"或者走向完全不同的逻辑 |
| `curl -o out.html https://a.com` | `-o` 在 Invoke-WebRequest 里没有对应语义 |
| `curl -w "%{http_code}"` | 直接报错 |
| `curl -u user:pass` | 不识别 |

而且它吐出来的错误信息是 PowerShell 的，不是 curl 的，排错时会把人带偏。

解决办法有三个，我选了第一个：

1. **显式写 `curl.exe`**，绕过别名，永远指向真正的二进制；
2. 用 `cmd` 而不是 PowerShell；
3. 在 Linux/macOS 上则完全没有这个问题。

确认版本：

```powershell
curl.exe -V
```

```
curl 8.22.0 (x86_64-w64-mingw32) libcurl/8.22.0 Schannel zlib/1.3.2 brotli/1.2.0 zstd/1.5.7 libidn2/2.3.8 libpsl/0.21.5 libssh2/1.11.1 WinLDAP
Release-Date: 2026-09-02
```

这一行信息量其实很大：

- `8.22.0`：curl 本体版本；
- `libcurl/8.22.0`：真正干活的是库；
- `Schannel`：Windows 上用的 TLS 后端（Linux 上一般是 OpenSSL），所以某些 TLS 报错的措辞会和网上教程不一样；
- `zlib/brotli/zstd`：支持的压缩算法，决定了 `--compressed` 能不能用；
- `libssh2`：支持 SFTP/SCP 的那部分。

> **本节结论**：本文后面所有命令一律写 `curl.exe`。你在 CMD 里照抄可以省略 `.exe`。

## 3. 在 Kali 里查命令用法

我在 Kali 的终端里启动了 PowerShell，直接输入了 `curl`。这次没有提供网址，终端提示可以用 `curl --help` 查看用法。

```powershell
curl --help
```

![Kali 中的 PowerShell 显示 curl 帮助和常用参数](/images/curl-http-practice/kali-help.png)

帮助里把 `-I` 和 `--head` 列在同一行。这就是下面查看响应头时用到的参数。

参数很多，但日常真正反复用的也就十几个。我的记法是按"改请求"和"改输出"分组：

- **改请求**：`-X`（方法）、`-H`（头）、`-d`（体）、`-u`（认证）、`-b`（cookie）、`-A`（User-Agent）；
- **改输出**：`-i`（带头）、`-I`（只要头）、`-o`（存文件）、`-s`（静默）、`-v`（详细过程）、`-w`（自定义格式）。

想看全部参数用 `curl --help all`，想看某个参数的完整说明查 [curl 手册](https://curl.se/docs/manpage.html)。

## 4. 在 Windows CMD 里查看响应头

```bat
curl -I https://www.baidu.com
```

![Windows CMD 运行 curl -I 后显示百度首页的响应头](/images/curl-http-practice/windows-head.png)

`-I` 是 `--head` 的简写。访问 HTTP 或 HTTPS 地址时，curl 会发送 HEAD 请求，终端显示状态行和响应头。HEAD 响应没有正文。[curl 手册](https://curl.se/docs/manpage.html#-I)

输出的第一行是 `HTTP/1.1 200 OK`。`HTTP/1.1` 是这次响应使用的协议版本，`200 OK` 表示服务器成功处理了请求。往下还能看到 `Content-Type: text/html`，它给出了服务器声明的内容类型。

截图里还有一行 `Content-Length: 0`。这次执行的是 HEAD，正文是否有内容要看另一次 GET 请求的返回。[HTTP 规范里的 HEAD 方法](https://www.rfc-editor.org/rfc/rfc9110.html#name-head)

在 PowerShell 里重跑一次，输出更整齐：

![](/images/curl-http-practice/ps-head.png)

逐行拆一下（这条命令是排查任何网站的第一步，值得记住）：

| 行 | 含义 | 能看出什么 |
| --- | --- | --- |
| `HTTP/1.1 200 OK` | 协议版本 + 状态码 + 原因短语 | 服务活着、请求合法 |
| `Server: bfe` | 服务器软件标识 | 百度自研的接入层，不是 nginx/Apache |
| `Content-Type: text/html` | 正文类型 | 浏览器据此决定怎么渲染 |
| `Content-Length: 0` | 正文长度 | HEAD 不返回正文，所以是 0 |
| `Cache-Control: private, no-cache...` | 缓存策略 | CDN/浏览器该怎么缓存 |
| `Date` | 服务器时间（GMT） | 和你本地时间差太多说明时钟有问题 |

> `-I`（大写 i）只要头，`-i`（小写 i）是头 + 正文一起打印。这两个参数只差一个大小写，效果完全不同，我一开始就搞混过。

## 5. 在 Kali 里读取 API 返回的正文

这次在 Kali 里换了一个地址，访问 GitHub 的 `/zen` 接口。它会返回一句简短的纯文本，适合直接在终端里看。[GitHub 接口说明](https://docs.github.com/en/rest/meta/meta?apiVersion=2022-11-28#get-the-zen-of-github)

```bash
curl https://api.github.com/zen
```

![Kali 中运行 curl 请求 GitHub 的 zen 接口并显示返回的英文句子](/images/curl-http-practice/kali-get.png)

这条命令只带了网址，curl 默认发送 GET 请求，把正文打印到终端。截图里的正文是 `Anything added dilutes everything else.`。截图中的提示符显示我当时在 root 环境里，普通用户也可以运行这条命令。[curl 手册](https://curl.se/docs/manpage.html)

这里有个细节：不加 `-s` 时，curl 会往 stderr 打一行进度条（`% Total % Received ...`）。在终端里看无所谓，但一旦要 `| grep` 或者写进脚本，就必须加 `-s`，否则进度条会污染管道。本文后面基本都带 `-s`。

## 6. 看一次请求的全过程：`-v`

前面两条命令都只给你结果。想知道"到底发了什么、收了什么"，用 `-v`。

```powershell
curl.exe -sv https://api.github.com/zen
```

![](/images/curl-http-practice/ps-verbose.png)

`-v` 的输出分三类行，看懂这三类就够用了：

| 前缀 | 含义 |
| --- | --- |
| `*` | curl 自己的过程信息：DNS、TCP 连接、TLS 握手、证书校验 |
| `>` | 发出去的请求头（`>` 表示 out） |
| `<` | 收到的响应头（`<` 表示 in） |

先看 `*` 这一段，它把一次 HTTPS 请求拆成了肉眼可见的步骤：

```
* Host api.github.com:443 was resolved.
* IPv4: 20.205.243.168
*   Trying 20.205.243.168:443...
* schannel: SSL/TLS connection with api.github.com port 443 (step 1/3)
* ALPN: curl offers http/1.1
* schannel: SSL/TLS handshake complete
* ALPN: server accepted http/1.1
* Established connection to api.github.com (20.205.243.168 port 443) from 192.168.1.195 port 18189
```

- `resolved` / `IPv4`：DNS 解析结果，注意 curl 会同时尝试 IPv6 和 IPv4；
- `Trying`：TCP 三次握手开始；
- `schannel ... (step 1/3)`：Windows 用的是 **Schannel**（系统自带的 TLS 实现），Linux 上这里会显示 OpenSSL，措辞完全不同；
- `ALPN: curl offers http/1.1` → `server accepted http/1.1`：双方协商协议版本。这台机器上的 curl 编译时没带 HTTP/2 支持，所以只报了 http/1.1；
- `Established connection ... from 192.168.1.195 port 18189`：连接建立，还顺便告诉你用了哪个本地端口。

**排错的顺序就藏在这几行里**：卡在 `resolved` 是 DNS 问题，卡在 `Trying` 是网络/防火墙问题，卡在 `schannel` 是证书/中间人的问题。看一眼就知道该往哪查。

从截图里能直接读到：

```
> GET /zen HTTP/1.1
> Host: api.github.com
> User-Agent: curl/8.22.0
> Accept: */*
```

- `GET /zen HTTP/1.1`：方法 + 路径 + 协议版本，注意路径不含域名，域名在 `Host` 里；
- `Host`：HTTP/1.1 唯一必需的请求头，一个 IP 上挂多个站点就靠它区分；
- `User-Agent: curl/8.22.0`：默认 UA，服务端一眼就能看出你不是浏览器（有些站点会因此返回不同内容，需要伪装时用 `-A`）。

响应头部分：

![](/images/curl-http-practice/ps-verbose2.png)

有意思的几行：

```
< HTTP/1.1 200 OK
< x-github-api-version-selected: 2022-11-28
< Access-Control-Allow-Origin: *
< Strict-Transport-Security: max-age=31536000; includeSubdomains; preload
< X-Frame-Options: deny
< X-Content-Type-Options: nosniff
< Server: github.com
< X-RateLimit-Remaining: 54
```

`x-github-api-version-selected` 说明服务端替你选了一个 API 版本；`X-RateLimit-Remaining: 54` 是未认证请求的额度（每小时 60 次），**这条 API 一旦写到脚本里，第一天就会被限流**；后四行是标准的安全响应头——强制 HTTPS、禁止被 iframe 嵌套、禁止 MIME 嗅探、限制内容来源。**做安全的人看响应头，看的就是这些。**

> `-v` 的输出走 stderr，正文走 stdout，所以 `curl -sv url > body.txt` 依然能干净地把正文存下来。

## 7. 把结果存成文件：`-o` / `-O` / `-D`

终端里刷屏几百 KB 的 HTML 没有意义。存下来。

```powershell
curl.exe -s -o baidu.html -w "code=%{http_code} size=%{size_download} time=%{time_total}s" https://www.baidu.com
```

![](/images/curl-http-practice/ps-save.png)

输出：

```
code=200 size=2443 time=0.297231s
```

`-w`（write out）是 curl 最被低估的参数：它让你在**不改变输出内容**的前提下，额外打印一段自定义信息。`%{http_code}`、`%{size_download}`、`%{time_total}` 这些变量叫"输出变量"，完整列表在 [curl 手册 -w](https://curl.se/docs/manpage.html#-w)。

几个容易混的参数：

| 参数 | 作用 | 典型场景 |
| --- | --- | --- |
| `-o name` | 存到指定文件名 | 脚本里最常用 |
| `-O` | 用 URL 里的文件名（大写 O） | 下载软件包 |
| `-J` | 用 `Content-Disposition` 里的文件名 | 配合 `-O` 下载附件 |
| `-D file` | 只把**响应头**写进文件 | 留证据 |
| `-i` | 头 + 正文一起打印到 stdout | 快速看全貌 |
| `-o NUL` | 丢掉正文（Windows 的"黑洞"） | 只关心状态码时 |

把头和正文分开存，是留证据的标准姿势：

```powershell
curl.exe -s -D headers.txt -o body.html https://www.baidu.com
```

![](/images/curl-http-practice/ps-split.png)

`headers.txt` 里就是完整的响应头，`body.html` 是 2443 字节的正文。事后复盘时这两个文件比终端截图有用得多。

## 8. 跟随跳转：`-L`

现在几乎每个站点都会跳转：`http → https`、`example.com → www.example.com`、短链、登录后回跳。默认情况下 **curl 不跟随跳转**，它会把 302 原样交给你。

```powershell
curl.exe -s -o NUL -w "no -L : %{http_code} -> %{redirect_url}" http://httpbin.org/redirect/1
curl.exe -sL -o NUL -w "with -L: %{http_code} -> %{redirect_url}" http://httpbin.org/redirect/1
```

![](/images/curl-http-practice/ps-redirect.png)

```
no -L : 302 -> http://httpbin.org/relative-redirect/1
with -L: 200 ->
```

两件事说清楚了：

1. 不带 `-L`，你拿到的是 `302`，`%{redirect_url}` 告诉你要跳去哪；
2. 带 `-L`，curl 自动跟到最终页面，状态码变成 `200`，`redirect_url` 为空（因为没有下一跳了）。

`-L` 还有两个配套参数：

- `--max-redirs N`：最多跟几次，防止被重定向环玩死；
- `--location-trusted`：跨域跳转时也把 `Authorization` 头带过去（默认不带，这是安全设计）。

> 排查"登录后一直跳回登录页"这类问题，第一步就是 `curl -svL` 看跳转链。浏览器把中间的 302 全藏起来了，curl 不会。

## 9. 提交数据：`-d` / `--data-binary` / `-H`

GET 是把参数塞在 URL 里，POST 是把数据放在请求体里。curl 用 `-d` 发送请求体，并且会自动把方法改成 POST。

这里有个 Windows 特有的坑：**PowerShell 会吃掉内联 JSON 里的双引号**。

```powershell
# 看起来没问题，实际发出去的 body 是坏的
curl.exe -s -X POST -H "Content-Type: application/json" -d '{"user":"0x1oly404"}' https://httpbin.org/post
```

PowerShell 在把参数传给原生命令时，会按自己的规则重新拼命令行，JSON 里的 `"` 经常被剥掉，服务端收到的就是 `{user:0x1oly404}` 这种非法 JSON（httpbin 的 `"json": null` 就是证据）。

正确做法是**把 body 写进文件，用 `--data-binary @文件`**：

```powershell
'{"user":"0x1oly404","action":"curl-practice"}' | Set-Content -Encoding ascii payload.json
curl.exe -s --data-binary "@payload.json" -H "Content-Type: application/json" https://httpbin.org/post
```

![](/images/curl-http-practice/ps-post.png)

httpbin 会把收到的请求原样回显，正好用来验证"服务端到底收到了什么"：

```json
{
  "method": "POST",
  "headers": {
    "Content-Length": "46",
    "Content-Type": "application/json",
    "User-Agent": "curl/8.22.0"
  },
  "json": {
    "action": "curl-practice",
    "user": "0x1oly404"
  }
}
```

`"json"` 字段不是 null，说明服务端成功把 body 解析成了 JSON。

`-d` 和 `--data-binary` 的区别值得记一下：

| 参数 | 行为 |
| --- | --- |
| `-d` / `--data` | 会去掉换行、把 `@file` 的内容当表单处理 |
| `--data-binary` | **原样**发送文件字节，不做任何处理 |
| `--data-urlencode` | 发送前做 URL 编码 |

传 JSON 一律用 `--data-binary`，否则文件里的换行会被悄悄删掉，某些签名接口直接报错。

## 10. Basic 认证：`-u`

```powershell
curl.exe -s -u admin:Sec@123 https://httpbin.org/basic-auth/admin/Sec%40123
```

![](/images/curl-http-practice/ps-auth.png)

```json
{
  "authenticated": true,
  "user": "admin",
  "authorized": true
}
```

`-u user:pass` 做的是 HTTP Basic 认证：把 `user:pass` 用 base64 编码后塞进 `Authorization: Basic xxx` 请求头。

> **base64 是编码，不是加密。** 抓包的人一眼就能还原出密码。所以 Basic 认证只有在 HTTPS 下才有意义，明文 HTTP 上等于把密码写在信封外面。

另外 `-u` 会把密码留在 shell history 和进程列表里。生产脚本里更稳的写法是：

```powershell
curl.exe -s -K curlrc.txt      # 把 user = "admin:pass" 写进配置文件
curl.exe -s --netrc-file .netrc
```

## 11. 带着 cookie 走：`-c` / `-b`

很多站点靠 Cookie 维持会话。curl 的做法是给一个"cookie 罐"（cookie jar）：

```powershell
curl.exe -s -c cookies.txt -b cookies.txt -L "https://httpbin.org/cookies/set?token=abc123&uid=1001"
```

![](/images/curl-http-practice/ps-cookie.png)

```json
{
  "cookies": {
    "token": "abc123",
    "uid": "1001"
  }
}
```

- `-c cookies.txt`：把响应里 `Set-Cookie` 下发的 cookie **写进**文件；
- `-b cookies.txt`：请求时把文件里的 cookie **带上**。

两个一起写，就完成了一次"登录 → 拿到会话 → 后续请求带上会话"的闭环。`cookies.txt` 是 Netscape 格式的纯文本，可以直接打开看，也可以手工改。

`-u` 和 `-b` 的区别：前者是"你是谁"（认证），后者是"你之前来过"（状态）。真实系统里通常两个都要。

## 12. 把 curl 当性能探针：`-w`

`-w` 不只能打状态码，还能打时间。这是我用得最多的一段：

```powershell
curl.exe -s -o NUL -w "dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s" https://www.baidu.com
```

![](/images/curl-http-practice/ps-timing.png)

```
dns=0.018032s connect=0.083861s tls=0.203772s ttfb=0.258335s total=0.258393s
```

这五个时间点是**累加**的，从请求开始算起。理解它们就能定位"慢在哪一层"：

| 变量 | 含义 | 数值大说明什么 |
| --- | --- | --- |
| `time_namelookup` | DNS 解析完成 | DNS 慢，换解析器或加缓存 |
| `time_connect` | TCP 三次握手完成 | 网络 RTT 高、链路差、被限速 |
| `time_appconnect` | TLS 握手完成 | 证书链长、TLS 版本低、服务端 CPU 忙 |
| `time_starttransfer` | 收到第一个字节（TTFB） | 服务端处理慢、数据库慢、后端排队 |
| `time_total` | 整个请求结束 | 正文大、带宽小 |

用差值看更准：

- `connect - namelookup` ≈ TCP 握手耗时 ≈ 1 个 RTT；
- `appconnect - connect` ≈ TLS 握手耗时 ≈ 1~2 个 RTT；
- `starttransfer - appconnect` ≈ **服务端真正处理请求的时间**；
- `total - starttransfer` ≈ 传正文的时间。

这次的数据拆开看：DNS 18ms、TCP 握手 66ms、TLS 握手 120ms、服务端处理 55ms。TLS 那一段比 TCP 还长，因为全握手要多跑两个来回（TLS 1.2）；TTFB 之后只用了 0.06ms 就把 2.4KB 的正文收完，说明瓶颈完全不在带宽。

> 线上排查"接口慢"，先跑这条命令。如果 TTFB 就很大，问题在后端，不在网络；如果 TTFB 正常但 total 很大，问题在响应体积或带宽。**先分层，再看代码。**

## 13. 批量状态码巡检

单个 URL 看不出问题，一批 URL 才能。这里用 `-o NUL` 丢掉正文，`-w` 只打状态码：

```powershell
"/","/admin/","/login.php","/.git/config","/backup.zip" | % { "$(curl.exe -s -o NUL -w '%{http_code}' "https://www.baidu.com$_")  $_" }
```

![](/images/curl-http-practice/ps-status.png)

```
200  /
404  /admin/
302  /login.php
404  /.git/config
404  /backup.zip
```

状态码是服务器对"你能不能看"的第一句回答，值得背下来：

| 状态码 | 含义 | 排查时的意义 |
| --- | --- | --- |
| 200 | 成功 | 资源存在且可读 |
| 301 / 302 | 永久 / 临时跳转 | 注意 `Location`，可能是登录墙 |
| 401 | 未认证 | 需要凭据，试试 `-u` |
| 403 | 已认证但无权限 | 存在但被拦，值得记录 |
| 404 | 不存在 | 路径错，或者被刻意隐藏 |
| 405 | 方法不允许 | 换个方法试试（`-X POST`） |
| 500 | 服务端错误 | 后端崩了，可能是输入触发的 |
| 502 / 504 | 网关错误 / 超时 | 反向代理后面的服务挂了 |

这条循环还有两个工程化细节：

1. `-o NUL` 一定要加，否则每个响应都会打到终端；
2. `-w '%{http_code}'` 里的单引号不能换成双引号，PowerShell 的双引号会先做变量展开，`%{...}` 会被当成变量。

> 这种巡检只对**自己有权限的站点**做。对别人的站点批量扫路径，性质就变了。

## 14. 超时与退出码：`-m` / `-sS`

网络请求必须设超时，否则脚本会挂死。

```powershell
curl.exe -sS -m 3 https://httpbin.org/delay/10; "exit=$LASTEXITCODE"
```

![](/images/curl-http-practice/ps-timeout.png)

```
curl: (28) Operation timed out after 3012 milliseconds with 0 bytes received
exit=28
```

三个知识点：

1. `-m 3` 是**整个请求**的总时限（`--max-time`），`--connect-timeout` 只管连接阶段，两个可以分开设；
2. `-sS` = 静默 + 静默但保留错误。只用 `-s` 会把错误信息也吞掉，排错时很坑；
3. **退出码**（`$LASTEXITCODE` / `$?`）才是脚本判断成败的依据，不是输出文本。

常见退出码：

| 退出码 | 含义 |
| --- | --- |
| 0 | 成功 |
| 6 | 域名解析失败 |
| 7 | 连接被拒绝（端口没开） |
| 28 | 超时 |
| 35 | TLS 握手失败 |
| 60 | 证书校验失败（自签名、过期、域名不匹配） |

脚本里的标准写法：

```powershell
curl.exe -sS -m 5 -o out.json https://api.example.com/health
if ($LASTEXITCODE -ne 0) { Write-Host "请求失败，退出码 $LASTEXITCODE" }
```

## 15. curl 常用参数速查表

| 分类 | 参数 | 说明 |
| --- | --- | --- |
| 方法 | `-X POST` | 指定 HTTP 方法 |
| 请求头 | `-H "K: V"` | 加自定义头，可重复 |
| 请求体 | `-d` / `--data-binary` | 表单 / 原样字节 |
| 上传 | `-F "file=@a.png"` | multipart 表单 |
| 认证 | `-u user:pass` | Basic 认证 |
| Cookie | `-b` / `-c` | 读 / 写 cookie 罐 |
| 输出 | `-o` / `-O` / `-D` | 存正文 / 用远端名 / 存头 |
| 只看头 | `-I` | HEAD 请求 |
| 头+体 | `-i` | 打印到终端 |
| 跳转 | `-L` | 跟随重定向 |
| 静默 | `-s` / `-sS` | 静默 / 静默但留错误 |
| 详细 | `-v` | 打印整个过程 |
| 计时 | `-w "%{time_total}"` | 自定义输出 |
| 超时 | `-m` / `--connect-timeout` | 总时限 / 连接时限 |
| TLS | `-k` | 跳过证书校验（**仅调试**） |
| 代理 | `-x http://127.0.0.1:8080` | 走代理 |
| 压缩 | `--compressed` | 声明支持压缩并自动解压 |
| 限速 | `--limit-rate 200k` | 限速下载 |

---

# 第二部分：nmap

## 16. nmap 是干什么的

curl 解决的是"**我已经知道地址了，去看看它返回什么**"。但真实场景里，第一步往往是"**这个网段里有哪些机器、它们开了哪些端口**"。这就是 `nmap`（Network Mapper）的活。

nmap 主要做四件事：

| 能力 | 参数 | 说明 |
| --- | --- | --- |
| 主机发现 | `-sn` | 这个网段里谁在线 |
| 端口扫描 | `-p` `-sT` `-sS` | 谁开了哪些端口 |
| 服务识别 | `-sV` | 端口后面跑的是什么软件、什么版本 |
| 脚本探测 | `--script` | NSE 脚本，做指纹、漏洞、枚举 |

两个工具的分工可以用一句话概括：

```
nmap  →  找到"门"（IP + 端口）
curl  →  走到门口看"门后面是什么"（HTTP 响应）
```

## 17. 安装与版本

Windows 上最省事的是 winget：

```powershell
winget install --id Insecure.Nmap -e
```

装完有两个坑：

**坑一：当前终端里 PATH 没刷新。** winget 装完不会更新已经开着的 PowerShell 会话，直接敲 `nmap` 会报"无法识别"。要么重开终端，要么临时补一下：

```powershell
$env:PATH += ";C:\Program Files (x86)\Nmap"
```

**坑二：中文 Windows 下时间戳乱码。** nmap 用 C 运行时的时区名，中文系统会输出 `?D1...` 这类乱码。设一个 `TZ` 环境变量就好：

```powershell
$env:TZ = 'CST-8'
```

确认安装：

```powershell
nmap --version
```

![](/images/curl-http-practice/nmap-version.png)

```
Nmap version 7.80 ( https://nmap.org )
Platform: i686-pc-windows-windows
Compiled with: nmap-liblua-5.3.5 openssl-1.0.2s nmap-libssh2-1.8.2 nmap-libz-1.2.11 nmap-libpcre-7.6 Npcap-0.9982 nmap-libdnet-1.12 ipv6
```

这里出现 `Npcap-0.9982` 是好事：Nmap 在 Windows 上依赖 **Npcap** 驱动才能发原始数据包（SYN 扫描、OS 识别都要它）。安装包里默认勾选，装完 `C:\Windows\System32\Npcap\` 下会出现 `Packet.dll` 之类的文件。

> 如果 `nmap -sS` 报 "Couldn't open a raw socket"，八成是没装 Npcap，或者没以管理员身份运行。

## 18. 主机发现：`-sn`

扫之前先知道谁在线。`-sn` 只做主机发现，不扫端口，速度快、动静小：

```powershell
nmap -sn 192.168.1.0/24
```

![](/images/curl-http-practice/nmap-sn.png)

```
Starting Nmap 7.80 ( https://nmap.org ) at 2026-11-09 21:12 CST
Nmap scan report for 192.168.1.1
Host is up (0.0020s latency).
MAC Address: 24:EF:B4:56:A8:96 (Unknown)
Nmap scan report for vivo-S17t.lan (192.168.1.67)
Host is up (0.0030s latency).
...
Nmap done: 256 IP addresses (4 hosts up) scanned in 18.06 seconds
```

这段输出信息很密，逐个看：

- `192.168.1.1` 没有主机名，延迟 2ms，还带了 MAC —— 这就是路由器/网关；
- 后面几个 `.lan` 域名是路由器 DHCP 分配的本地名字，说明 nmap 顺带做了反向解析；
- `MAC Address` 只在**同一广播域**里能拿到（局域网内 ARP 直接问），跨路由就没了；
- 最后一行 `256 IP addresses (4 hosts up)` 是结论：这个网段只有 4 台设备在线。

`-sn` 和"直接扫端口"的区别很大：

| 用法 | 发了什么 | 适用场景 |
| --- | --- | --- |
| `nmap -sn 网段` | ARP（局域网）/ ICMP / TCP ping | 先摸清有多少目标 |
| `nmap 网段` | 对每台在线的机器扫 1000 个常用端口 | 慢，且噪音大 |
| `nmap -Pn 目标` | 跳过主机发现，直接假定在线 | 目标禁 ping 时 |

> 家庭网络里扫出 4 台设备：路由器、一台手机、两台笔记本。这个结果和实际情况完全对得上——**能验证的结果才是可信的结果**。

## 19. 端口扫描：`-sT` 与 `-sS`

知道谁在线之后，看它开了哪些门。

```powershell
nmap -sT -p 22,80,443 scanme.nmap.org
```

![](/images/curl-http-practice/nmap-port.png)

```
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.23s latency).

PORT    STATE    SERVICE
22/tcp  open     ssh
80/tcp  open     http
443/tcp filtered https
```

`STATE` 这一列是重点，它只有三种主要取值：

| 状态 | 含义 | 怎么来的 |
| --- | --- | --- |
| `open` | 端口开着，有程序在监听 | 收到了 SYN-ACK |
| `closed` | 端口没开，但主机可达 | 收到了 RST |
| `filtered` | 不知道开没开，被拦了 | 没有响应，或 ICMP 不可达 |

这次 `443` 是 `filtered`，说明包发出去没人回——可能是防火墙丢包，也可能是云厂商的安全组。**"filtered" 不等于"关闭"，这一点很多人搞混。**

`-sT` 和 `-sS` 的区别是扫描原理：

| 参数 | 原理 | 特点 |
| --- | --- | --- |
| `-sT`（TCP connect） | 完整三次握手，连上就断 | 不需要特权，但会在目标日志里留下完整连接记录 |
| `-sS`（SYN 半开） | 只发 SYN，收到 SYN-ACK 就发 RST | 快、隐蔽，但需要 root/管理员 + Npcap |

Windows 上如果没有管理员权限，`-sS` 会报错，这时退回 `-sT` 就行。**能用、结果对，比"更高级"重要。**

## 20. 认服务：`-sV`

端口号只是约定，不是事实：80 端口上完全可以跑 SSH，8080 上可能是个 Java 应用。`-sV` 会真的去和端口对话，读 banner、发探针，判断软件和版本：

```powershell
nmap -sV -p 22,80 scanme.nmap.org
```

![](/images/curl-http-practice/nmap-sv.png)

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

三个信息点：

1. **`OpenSSH 6.6.1p1`**：SSH 的精确版本，附带 `Ubuntu 2ubuntu2.13` 这种发行版补丁号；
2. **`Apache httpd 2.4.7`**：Web 服务器软件和版本；
3. **`CPE: cpe:/o:linux:linux_kernel`**：标准化命名，可以直接拿去和 CVE 库比对。

从安全角度看，`-sV` 才是真正有价值的扫描——**版本号就是攻击面**。一个 2014 年的 Apache 2.4.7 意味着什么，查一下 CVE 就知道了。

`-sV` 比纯端口扫描慢很多，因为它要为每个开放端口跑一轮探针。这次两个端口花了 26 秒，而纯 `-sT` 只用了 14 秒。

## 21. 只看开着的：`--open` 与端口选择

真实扫描里绝大部分端口是关的，输出全是噪音。`--open` 只打印 open 的：

```powershell
nmap -sT --open -p 21,22,23,53,80,443,3306,3389,8080 scanme.nmap.org
```

![](/images/curl-http-practice/nmap-open.png)

```
Not shown: 7 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

9 个端口，7 个 filtered，2 个 open，输出干净了十倍。

我常用的端口清单（按用途记，比背数字有用）：

| 用途 | 端口 |
| --- | --- |
| Web | 80, 443, 8080, 8443 |
| 远程管理 | 22 (SSH), 23 (Telnet), 3389 (RDP), 5900 (VNC) |
| 文件共享 | 21 (FTP), 445 (SMB), 139 (NetBIOS), 2049 (NFS) |
| 数据库 | 3306 (MySQL), 5432 (PostgreSQL), 6379 (Redis), 27017 (MongoDB) |
| 邮件 | 25, 110, 143, 465, 587, 993 |

两个提速参数：

- `-T4`：加快时序模板（0~5，默认 3）。跨公网用 4 可以，再高容易丢包反而更慢；
- `--min-rate 1000`：每秒至少发 1000 个包，比 `-T4` 更直接。

全端口扫描是 `-p-`（等价于 `-p 1-65535`），但要注意：**对公网目标扫全端口非常显眼，也很慢**。我只对自己有权限的资产这么干，而且会配 `--min-rate` 和 `-oN` 留档。

> 顺便一个教训：第一次我对 `scanme.nmap.org` 跑了 `-p 1-1000 -T4`，跑了 5 分多钟，998 个端口全是 filtered。**公网主机前面的防火墙会让"全端口扫描"变成纯粹的等待**。后来改成精选端口列表，13 秒出结果。

## 22. NSE 脚本：`--script`

NSE（Nmap Scripting Engine）是 nmap 最强大的部分。脚本用 Lua 写，内置几百个，按类别分组：

| 类别 | 用途 |
| --- | --- |
| `default` | 默认跑的一批（`-sC` 等价于 `--script default`） |
| `safe` | 不会造成破坏的探测 |
| `auth` | 认证相关（弱口令、匿名登录） |
| `vuln` | 漏洞检测 |
| `discovery` | 信息收集（标题、目录、服务枚举） |
| `brute` | 暴力破解 |

拿 Web 指纹举例：

```powershell
nmap -sV --script http-title -p 80 scanme.nmap.org
```

![](/images/curl-http-practice/nmap-nse.png)

```
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Go ahead and ScanMe!
```

`|_` 开头是 NSE 的输出格式。这一条命令同时拿到了 Web 服务器版本和页面标题——**这就是"指纹"**：知道对方跑 Apache 2.4.7 on Ubuntu，后续该查什么漏洞、用什么 payload，方向就定了。

其他常用的：`http-headers`、`http-enum`（目录枚举）、`ssl-cert`（证书信息）、`ssh-auth-methods`。

> `vuln` 类脚本要谨慎用，它们会真的去触发漏洞。只在授权目标上用。

## 23. 保存结果：`-oN` / `-oX` / `-oG`

扫描结果必须落盘，否则过半小时你就想不起来当时扫的是哪台机器。

```powershell
nmap -sT -p 22,80 -oN scanme.txt scanme.nmap.org
Get-Content scanme.txt -TotalCount 11
```

![](/images/curl-http-practice/nmap-on.png)

```
# Nmap 7.80 scan initiated Mon Nov 09 21:12:31 2026 as: "C:\\Program Files (x86)\\Nmap\\nmap.exe" -sT -p 22,80 -oN scanme.txt scanme.nmap.org
Nmap scan report for scanme.nmap.org (45.33.32.156)
Host is up (0.23s latency).

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
# Nmap done at Mon Nov 09 21:12:40 2026 -- 1 IP address (1 host up) scanned in 9.54 seconds
```

注意第一行和最后一行：**nmap 自动把完整命令行和时间戳写进了结果文件**。这就是为什么它比自己 `>` 重定向要好——事后能完整复现。

三种输出格式：

| 参数 | 格式 | 用途 |
| --- | --- | --- |
| `-oN` | 普通文本 | 人看，最常用 |
| `-oX` | XML | 喂给其他工具、导入数据库 |
| `-oG` | grepable | 一行一台主机，适合 `grep`/`awk` |
| `-oA 前缀` | 三种全存 | 归档标准做法 |

## 24. 把两个工具串起来

单独用哪个都不完整。真实的排查顺序是：

```
nmap -sn        谁在线
   ↓
nmap -sT -p     开了哪些端口
   ↓
nmap -sV        端口后面是什么软件、什么版本
   ↓
curl -I         这个 Web 服务返回什么头
   ↓
curl -v / -w    细看请求过程与耗时
```

前面 nmap 已经确认 `scanme.nmap.org` 的 80 端口开着，并且是 Apache 2.4.7。现在用 curl 去敲门：

```powershell
curl.exe -sI http://scanme.nmap.org
```

![](/images/curl-http-practice/nmap-curl.png)

```
HTTP/1.1 200 OK
Date: Mon, 09 Nov 2026 13:02:00 GMT
Server: Apache/2.4.7 (Ubuntu)
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Type: text/html
```

两边对上了：`Server: Apache/2.4.7 (Ubuntu)` 和 nmap `-sV` 的结论完全一致。

**这就是两个工具的正确关系**：nmap 的 `-sV` 是"从外部猜"，curl 的响应头是"服务自己说的"。两个独立来源互相印证，结论才可信。如果 `-sV` 说是 nginx 而 `Server` 头说是 Apache，那多半中间有反向代理或者 WAF。

再往前走一步，可以用 PowerShell 把两步串成一条流水线（先确认端口，再取指纹）：

```powershell
$open = nmap -sT --open -p 80,443 -oG - target.example.com | Select-String "open"
if ($open) { curl.exe -sI http://target.example.com }
```

## 25. 边界：这些命令能对谁用

技术是中性的，用在哪有区别。我的原则很简单：

1. **自己的资产**：家里的路由器、自己的 VPS、自己的虚拟机，随便扫；
2. **明确授权的靶标**：`scanme.nmap.org` 是 Nmap 官方提供的练习站点，官方文档写明了允许扫描，但也请求"别太频繁"；
3. **别人的主机**：不管技术上行不行，未授权扫描在大多数司法辖区都属于违法行为，而且很容易被 IDS 记录下来。

还有两条实践习惯：

- **控制频率**：`-T4`、`--min-rate` 这类参数对线上服务是有压力的，公共靶标上我更愿意用默认 `-T3`；
- **留下记录**：`-oN` 存档，既是复盘材料，也是"我确实只扫了这些"的证据。

## 26. 复盘

这一圈下来，我最大的收获不是记住了多少个参数，而是理清了三层模型：

| 层 | 关心什么 | 工具 |
| --- | --- | --- |
| 主机层 | 哪些 IP 活着 | `nmap -sn` |
| 传输层 | 哪些端口开着、什么状态 | `nmap -sT/-sS -p` |
| 应用层 | 端口后面是什么服务、返回什么 | `nmap -sV`、`curl` |

排查问题时，**从上往下走，每一层只回答一个问题**：

- 连不上 → 先 `nmap -sn` 看主机在不在，再 `-p` 看端口通不通，最后 `curl -v` 看应用层；
- 慢 → `curl -w` 分段计时，先判断是 DNS、网络、TLS 还是后端；
- 返回不对 → `curl -v` 看请求头对不对，再看状态码。

工具只是手，模型才是脑。

几个还没解决的问题，留作下一步：

- 这次没能在 Windows 上跑 `-sS`（权限问题），下次在 Kali 里补上对比实验；
- `-O`（操作系统识别）没试，需要 Npcap + 管理员；
- NSE 的 `http-enum` 和 `vuln` 类脚本还没系统跑过。

---

## 附：全部命令清单

```powershell
# ---------- curl ----------
Get-Command curl | Format-Table CommandType, Name, Definition -AutoSize
curl.exe -V
curl --help

curl.exe -sI https://www.baidu.com
curl.exe -sv https://api.github.com/zen
curl.exe -s -o baidu.html -w "code=%{http_code} size=%{size_download} time=%{time_total}s" https://www.baidu.com
curl.exe -s -D headers.txt -o body.html https://www.baidu.com

curl.exe -s -o NUL -w "no -L : %{http_code} -> %{redirect_url}" http://httpbin.org/redirect/1
curl.exe -sL -o NUL -w "with -L: %{http_code} -> %{redirect_url}" http://httpbin.org/redirect/1

curl.exe -s --data-binary "@payload.json" -H "Content-Type: application/json" https://httpbin.org/post
curl.exe -s -u admin:Sec@123 https://httpbin.org/basic-auth/admin/Sec%40123
curl.exe -s -c cookies.txt -b cookies.txt -L "https://httpbin.org/cookies/set?token=abc123&uid=1001"

curl.exe -s -o NUL -w "dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s" https://www.baidu.com
curl.exe -sS -m 3 https://httpbin.org/delay/10

# ---------- nmap ----------
winget install --id Insecure.Nmap -e
$env:PATH += ";C:\Program Files (x86)\Nmap"
$env:TZ = 'CST-8'
nmap --version

nmap -sn 192.168.1.0/24
nmap -sT -p 22,80,443 scanme.nmap.org
nmap -sV -p 22,80 scanme.nmap.org
nmap -sT --open -p 21,22,23,53,80,443,3306,3389,8080 scanme.nmap.org
nmap -sV --script http-title -p 80 scanme.nmap.org
nmap -sT -p 22,80 -oN scanme.txt scanme.nmap.org

# ---------- 串联 ----------
curl.exe -sI http://scanme.nmap.org
```
