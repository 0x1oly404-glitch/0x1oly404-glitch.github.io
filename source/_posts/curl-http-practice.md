---
title: 用 curl 查看 HTTP 响应头和正文
date: 2026-11-09 20:10:00
slug: curl-http-practice
tags:
  - curl
  - HTTP
categories:
  - 网络实验
---

这次练习用到了两个终端。我在 Kali 虚拟机里查了 curl 的参数，在 Windows 的 CMD 里向百度首页发了一次 HEAD 请求，拿到的状态码是 `200 OK`。

<!-- more -->

## 在 Kali 里查命令用法

我在 Kali 的终端里启动 PowerShell 后，直接输入了 `curl`。这次没有提供网址，终端提示可以用 `curl --help` 查看用法。

```powershell
curl --help
```

![Kali 中的 PowerShell 显示 curl 帮助和常用参数](/images/curl-http-practice/kali-help.png)

帮助里把 `-I` 和 `--head` 列在同一行。这就是下面查看响应头时用到的参数。

## 在 Windows CMD 里查看响应头

```bat
curl -I https://www.baidu.com
```

![Windows CMD 运行 curl -I 后显示百度首页的响应头](/images/curl-http-practice/windows-head.png)

`-I` 是 `--head` 的简写。访问 HTTP 或 HTTPS 地址时，curl 会发送 HEAD 请求，终端显示状态行和响应头。HEAD 响应没有正文。[curl 手册](https://curl.se/docs/manpage.html#-I)

输出的第一行是 `HTTP/1.1 200 OK`。`HTTP/1.1` 是这次响应使用的协议版本，`200 OK` 表示服务器成功处理了请求。往下还能看到 `Content-Type: text/html`，它给出了服务器声明的内容类型。

截图里还有一行 `Content-Length: 0`。这次执行的是 HEAD，正文是否有内容要看另一次 GET 请求的返回。[HTTP 规范里的 HEAD 方法](https://www.rfc-editor.org/rfc/rfc9110.html#name-head)

## 在 Kali 里读取 API 返回的正文

这次在 Kali 里换了一个地址，访问 GitHub 的 `/zen` 接口。它会返回一句简短的纯文本，适合直接在终端里看。[GitHub 接口说明](https://docs.github.com/en/rest/meta/meta?apiVersion=2022-11-28#get-the-zen-of-github)

```bash
curl https://api.github.com/zen
```

![Kali 中运行 curl 请求 GitHub 的 zen 接口并显示返回的英文句子](/images/curl-http-practice/kali-get.png)

这条命令只带了网址，curl 默认发送 GET 请求，把正文打印到终端。截图里的正文是 `Anything added dilutes everything else.`。截图中的提示符显示我当时在 root 环境里，普通用户也可以运行这条命令。[curl 手册](https://curl.se/docs/manpage.html)
