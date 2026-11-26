# CTF 练习：HTTP 流量分析（webshell 取证）

> 平台：CTF² / 配套练习（[9.3.1 HTTP流量分析]，1 分，193 人解出）
> 环境：Kali Linux + Wireshark / tshark
> 日期：2026-11-11
> 性质：平台提供的授权靶场

---

## 一、目标

给定一个 24KB 的 `atta.pcapng` 流量包，从中还原出 flag。

## 二、附件

| 项 | 值 |
|---|---|
| 文件 | `atta.pcapng`（24 KB，137 个包） |
| MD5 | `d3b2f0a5cf87f23a74b144940c302948` |

> 平台下载时会重命名成长串哈希，用 `*.pcapng` 通配符即可。

## 三、方法一：strings 直接搜（最快）

```bash
md5sum ~/Downloads/*.pcapng
strings ~/Downloads/*.pcapng | grep -i flag
```

输出：

```
XaYflag.txt
XaYflag{This_is_a_f10g}
```

**10 秒出结果。** 简单流量题第一步永远是 `strings`。

## 四、方法二：Wireshark 定位（学取证）

1. 过滤 `http`，能看到一串对 `/shell.php`、`/shell1.php` 的请求
2. 用过滤器直接定位：

```
frame contains "flag{"
```

3. 命中第 105 号包（HTTP 200，text/html，5 行）
4. 展开 **Packet Details → Line-based text data**：

```
X@Yflag{This_is_a_f10g}
[S]\n
/var/www/html\n
[E]\n
X@Y
```

> `X@Y ... [S] ... [E] ... X@Y` 是 webshell 的输出分隔符：`[S]` 是命令输出，`[E]` 是错误输出。

命令行版：

```bash
tshark -r atta.pcapng -Y 'frame contains "flag{"' -T fields -e http.file_data
```

## 四·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先 `strings | grep flag`**

- 我在想：24KB 的小包，明文 HTTP，flag 很可能直接躺在载荷里。
- 验证：10 秒内搜到 `flag{This_is_a_f10g}`。
- 所以：**小包先 strings，不要一上来就 Wireshark 逐包点**。

**第 2 步：为什么还要用 Wireshark 复核**

- 我在想：strings 只能证明「字符串在里面」，不能说明它属于哪个请求、怎么来的。
- 验证：`frame contains "flag{"` 命中第 105 号 HTTP 200。
- 所以：能定位到具体会话，理解攻击链。

**第 3 步：为什么要看 `[S]`/`[E]` 结构**

- 我在想：这是 webshell 的输出分隔，说明响应是命令执行结果。
- 验证：第 105 包里同时有命令输出（flag、`/var/www/html`）和错误段。
- 所以：攻击者用 webshell 执行了 `ls` 和 `cat flag.txt`。

**第 4 步：为什么还原整条攻击链**

- 我在想：取证题的价值不只在 flag，而在「谁、怎么进来的、干了什么」。
- 验证：`POST /shell1.php` + 伪造 XFF/UA + 混淆载荷。
- 所以：能写出完整的入侵时间线。

### 命令是怎么来的

- **`strings pcap | grep -i flag`**：快速筛明文。
- **`tshark -Y 'frame contains "flag{"'`**：按内容过滤，脚本化复现。
- **为什么带 `http.file_data`**：直接取响应正文，不用手工点包。

### 逐段解释

| 动作 | 作用 |
|---|---|
| `md5sum` | 校验附件没下错 |
| `strings \| grep` | 快速定位明文 flag |
| `http` 过滤 | 列出所有请求 |
| `frame contains "flag{"` | 按内容精确定位 |
| Follow HTTP Stream | 看完整请求/响应 |

### 失败尝试与坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 下载的文件名是长哈希 | 平台重命名 | 用 `*.pcapng` 通配 |
| 手工翻包很慢 | 137 个包挨个点 | 先 strings，再 frame contains |

### 自测

**Q1**：为什么小流量包优先 `strings`？
**A**：明文协议下字符串直接可见，几秒出结果。

**Q2**：`[S]` / `[E]` 是什么？
**A**：webshell 输出分隔符，分别代表标准输出和错误输出。

**Q3**：流量分析对取证的意义？
**A**：即使日志被清，抓包仍能还原攻击者的每一步。

## 五、攻击链还原

从流量里能拼出完整的入侵过程：

| 元素 | 内容 |
|---|---|
| 请求 | `POST /shell1.php` |
| 伪造 IP | `X-Forwarded-For: 44.146.238.198` |
| 伪装 UA | `User-Agent: ... Baiduspider/2.0 ...` |
| 混淆载荷 | `pass=array_map("ass"."ert", array("ev"."al("...")))` |
| 先执行 | `ls` → 看到 `flag.txt` |
| 再执行 | `cat flag.txt` → 得到 flag |

**特点**：攻击者伪造了来源 IP 和 UA，用混淆的 PHP 一句话木马执行系统命令。

## 六、流量分析套路

| 步骤 | 动作 |
|---|---|
| 1 | `strings pcap \| grep -i flag` |
| 2 | Wireshark 过滤 `http`，看请求列表 |
| 3 | `frame contains "flag"` 直接定位 |
| 4 | 右键 → Follow → HTTP/TCP Stream 看完整会话 |
| 5 | 关注 POST 参数、响应正文、上传文件 |

## 七、修复与启示

| 问题 | 修复 |
|---|---|
| 存在 `shell1.php` | 文件上传白名单 + 禁止解析 + 目录外存储 |
| 默认 UA/IP 可伪造 | 不能作为访问控制依据 |
| 命令执行日志 | 记录并告警 Web 目录下的异常命令调用 |

**流量分析的意义**：即使服务器日志被清，抓包也能完整还原攻击者的每一步。

## 八、收获

- 学会 `strings` 快速搜流量包。
- 掌握 `frame contains` 这种「按内容过滤」的定位方法。
- 看懂了一个 webshell 的输出格式（`[S]`/`[E]` 分隔）。
- 从一条 flag 扩展到「还原整条攻击链」的思路。
