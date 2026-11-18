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
