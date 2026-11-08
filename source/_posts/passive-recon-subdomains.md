---
title: 网络空间搜索引擎与子域名收集实验
date: 2026-11-08 18:53:23
slug: passive-recon-subdomains
tags:
  - ZoomEye
  - Shodan
  - 信息收集
  - 子域名
categories:
  - 网络实验
---

这次作业先对照了几家平台的官方语法说明，再以 `iana.org` 为例查询公开记录。实际操作中，Shodan 的文本搜索取得了结果，证书透明度日志也返回了数据。ZoomEye 和部分筛选功能遇到了访问限制，下面保留这些实际情况。

实验日期为 2026 年 11 月 8 日，时间按北京时间记录。查询范围限于第三方已有的公开索引，以及少量正常 DNS 查询。

<!-- more -->

## 与 ZoomEye 功能相近的搜索引擎

ZoomEye 可以按域名、端口和服务特征寻找互联网资产。比如，查询一个域名关联的记录后，可以继续查看平台采集到的协议、网页标题和证书信息。根据它的官方说明，当前域名检索可以使用 `domain="iana.org"`。[ZoomEye 官方语法说明](https://github.com/zoomeye-ai/ZoomEye-python/blob/master/api.md)

同类工具里，下面几家都可以用于这次作业。表中的侧重点用于区分它们提供的查询视角，数据覆盖和账号权限仍要看实际页面。

| 平台 | 可以查询的主要信息 | 在这次作业中的用途 |
| --- | --- | --- |
| [FOFA](https://fofa.info/api) | 域名、IP、端口，网页标题与响应内容等 | 用域名条件寻找关联的 Web 记录，再查看服务信息 |
| [Shodan](https://help.shodan.io/the-basics/search-query-fundamentals) | 联网服务的响应内容，及端口、组织等属性 | 阅读服务响应，检查一条搜索结果为什么会命中 |
| [Censys](https://docs.censys.com/docs/censys-query-language) | 主机、Web 资产和证书记录 | 从主机名与证书两个方向核对候选资产 |
| [奇安信 HUNTER](https://www.qianxin.com/product/detail/pid/446) | 域名、IP、应用识别及备案关联等 | 从域名或企业信息继续查找关联资产 |
| [360 Quake](https://security.360.cn/Product/productDetail/id/296) | 网络空间中的资产与服务信息 | 作为另一个测绘数据来源，补充交叉查询 |

这里还要分清两种工具。`crt.sh` 主要提供证书透明度日志查询，Subfinder 主要从被动数据源收集子域名。它们适合补充域名线索，与上面几家平台提供的服务测绘信息各有用途。

![图 1　从公开线索到 DNS 核验的实验过程示意](/images/passive-recon/01-workflow.svg)

## 平台检索怎样做，实际取得了什么

### ZoomEye 的域名条件

打开 ZoomEye，在搜索框中输入下面的条件。

```text
domain="iana.org"
```

如果需要继续限定 HTTPS 常用端口，可以使用官方语法中的端口条件。

```text
domain="iana.org" && port=443
```

本次访问第一条查询对应的页面时，浏览器收到 HTTP 403 响应。下面是实际截图，未取得可核实的资产结果，因此不填写 ZoomEye 的命中数量。第二条只作为组合条件的写法示例。

![图 2　本次 ZoomEye 域名查询页面返回 403，截图记录访问结果](/images/passive-recon/02-zoomeye-access.png)

FOFA 可以按官方域名字段编写 `domain="iana.org"` 进行对照查询。本次浏览器中的 FOFA 结果页未成功加载，也没有取得可记录的命中数。

### Shodan 的字段搜索与文本搜索

Shodan 的字段搜索可以写成下面这样。

```text
hostname:iana.org
```

本次未登录账号，页面明确提示需要登录才能使用搜索筛选条件。

![图 3　Shodan 对 hostname 条件显示登录提示](/images/passive-recon/03-shodan-filter.png)

随后改用公开文本搜索，输入带英文双引号的域名文字。

```text
"www.iana.org"
```

这次成功返回了结果页，截图中显示 357 条匹配记录。[本次文本查询入口](https://www.shodan.io/search?query=%22www.iana.org%22)

![图 4　Shodan 文本搜索的实际结果，左侧显示 357 条匹配记录](/images/passive-recon/04-shodan-public.png)

这张图里有一个很适合检查的细节。第一条结果展示了 IANA 相关主机的证书和 HTTP 响应，响应状态为 `503 Service Unavailable`。往下还有一条来自其他网络的记录，其响应文字中包含 `http://www.iana.org`。

Shodan 官方说明中写明，默认搜索针对服务的响应内容。于是，一段响应只要带有这个网址，也可能被这条文本查询找到。357 条表示本次查询的文本匹配数量，不能直接写成 IANA 拥有 357 台服务器，也不能直接当成 357 个子域名。

判断一条记录是否与目标有关，还要继续看主机名和证书，结合 DNS 核验归属。平台给出的采集时间也应保留，截图中的响应状态只能说明它采集那次的情况。

### Censys 的语法需要看当前版本

查阅 Censys 的官方文档时，可以看到当前 Platform 使用 CenQL。查询一个明确的 Web 主机名，可以写成下面这样。

```text
web.hostname="www.iana.org"
```

这条是依据官方字段说明整理的查询示例，本次没有登录执行，也没有列出对应结果。复现时应使用当前 Platform 的字段提示，旧版教程中的写法需要先检查是否兼容。[Censys 新旧语法转换说明](https://docs.censys.com/docs/query-converter)

## 从证书日志收集子域名

资产搜索里的一个域名经常对应多条端口或服务记录。要整理子域名清单，还需要把名称单独提取出来。

本次使用 `iana.org` 作为范围。`www.iana.org` 属于它的子域名，`handbook.int.iana.org` 也属于这个范围。筛选时要保留完整名称，不能把多级子域名截成同一个名字。

证书透明度机制会把证书及预证书加入可公开检查的日志。证书中出现过的名称，可以成为子域名收集的线索。[证书透明度的工作方式](https://certificate.transparency.dev/howctworks/)

在 `crt.sh` 中查询 `%.iana.org`，也可以直接取得 JSON 数据。下面这条 PowerShell 命令把返回内容保存到本地。

```powershell
Invoke-RestMethod 'https://crt.sh/?q=%25.iana.org&output=json' |
  ConvertTo-Json -Depth 6 |
  Set-Content -LiteralPath 'ct.json' -Encoding utf8
```

网址中的 `%25` 是百分号的 URL 编码。若网页暂时无法加载，可以稍后重试同一个公开接口。本次浏览器访问查询页出现过网关错误，最终通过公开 JSON 接口取得了数据。

接着用 Python 读取每条记录的 `name_value`。这个字段里可能有换行，同一个名称也可能随证书续期反复出现，因此需要拆行和去重。

```python
import json
from pathlib import Path

records = json.loads(Path("ct.json").read_text(encoding="utf-8-sig"))
names = set()
for record in records:
    for value in record.get("name_value", "").splitlines():
        name = value.strip().lower().rstrip(".")
        if name.endswith(".iana.org") and "*" not in name:
            names.add(name)

Path("subdomains.txt").write_text(
    "\n".join(sorted(names)) + "\n", encoding="utf-8"
)
print(len(records), len(names))
```

本次原始响应有 276 条记录。按上述规则处理后，得到 24 个不同的、没有通配符的候选子域名。

![图 5　依据本次原始 JSON 生成的本地整理页截图，展示全部 24 个候选名称](/images/passive-recon/05-ct-results.png)

图 5 是本地整理页的截图，保留了来源网址与处理规则。它展示的是已保存的真实数据，页面样式由本次实验制作。

原始数据还出现了 `*.iana.org` 和 `*.int.iana.org`。这类通配符证书不能列举它覆盖范围里的每一个具体主机名，因此单独保留，不把星号替换成猜测的名称。根域名 `iana.org` 也没有计入这 24 个子域名。

276 条证书相关记录与 24 个候选名称采用不同的计数方式。同一个名称多次出现在日志里，去重后只计一次。证书记录还可能保留已经停用的名称，所以这份清单需要继续核验。

## 用 DNS 核验候选名称

本次从清单中抽查了六个名称，分别查询 A 和 AAAA 记录。A 记录给出 IPv4 地址，AAAA 记录给出 IPv6 地址；查询过程中还可能遇到指向另一个名称的 CNAME。

Windows 上可以使用下面两条命令复查。

```powershell
Resolve-DnsName -Name 'www.iana.org' -Type A -DnsOnly
Resolve-DnsName -Name 'www.iana.org' -Type AAAA -DnsOnly
```

![图 6　依据本次 DNS 原始输出生成的本地整理页截图，展示六个候选的抽查结果](/images/passive-recon/06-dns-results.png)

这六个候选中，五个在本次查询中取得了最终 IP 地址。`int.iana.org` 的查询只记录到一个 CNAME，没有取得最终 A 或 AAAA 地址。这里把它保留为待核验项，不根据这一次结果判断网站是否正常运行。

`www.iana.org`、`data.iana.org` 和 `pen.iana.org` 都经过各自的 CNAME，解析到 `104.18.24.232` 与 `104.18.25.232` 这两个 IPv4 地址。图 7 把这次观察到的关系画了出来。

![图 7　三组 CNAME 指向同一对 Cloudflare IPv4 地址，依据本次 DNS 输出绘制](/images/passive-recon/07-cname.svg)

这种情况说明，按 IP 去重会丢掉不同的域名名称。共同使用一对地址，也不足以证明这些地址上的其他网站都属于 IANA。整理清单时，应保留域名、CNAME 链和地址之间的对应关系。

`ftp.iana.org` 本次解析到 `ianaftp.vip.icann.org`，得到 IPv4 地址 `192.0.32.8`。CNAME 目标已经在 `iana.org` 范围之外，记录这条解析关系即可，不把外部域名自动加入子域名清单。

## 怎样继续补充这份清单

本次得到的 24 个名称只覆盖这次证书日志响应里出现的内容。还可以从普通搜索引擎收录的网页链接寻找线索，或者使用 Subfinder 查询它支持的被动数据源。

```text
site:iana.org -site:www.iana.org
```

```shell
subfinder -d iana.org -silent -o subfinder.txt
```

上面两条是后续补充方法，本次没有执行。Subfinder 官方说明将它定位为被动子域名发现工具；部分数据源需要相应配置，实际返回内容应以运行结果为准。[Subfinder 官方说明](https://docs.projectdiscovery.io/opensource/subfinder/overview)

从不同来源拿到名称后，可以沿用本文的拆行、范围筛选和去重规则，再对新增名称查询 DNS。只有六个名称接受过本次抽查，其余候选仍保留证书日志线索的状态。

复查时可以下载这次保存的[证书日志原始响应](/downloads/passive-recon/ct-iana.json)、[24 个候选名称](/downloads/passive-recon/subdomains.txt)和[DNS 抽查原始输出](/downloads/passive-recon/dns-iana.json)。再次查询时，保留新的时间与响应文件，就能具体比较哪些名称发生了变化。

## 参考资料

- [ZoomEye 官方 API 与检索语法](https://github.com/zoomeye-ai/ZoomEye-python/blob/master/api.md)
- [Shodan 查询与筛选说明](https://help.shodan.io/the-basics/search-query-fundamentals)
- [Censys 当前查询语言](https://docs.censys.com/docs/censys-query-language)
- [证书透明度机制说明](https://certificate.transparency.dev/howctworks/)
- [Subfinder 官方说明](https://docs.projectdiscovery.io/opensource/subfinder/overview)
