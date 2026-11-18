# DVWA XSS（反射型，Low）从弹窗到 Burp 抓包

> 靶场：DVWA / XSS (Reflected)，安全等级 Low
> 环境：Kali Linux 2026.2 + PHP 8.4.24 内置服务器（8080）+ Burp Suite 2026.3（8081）
> 日期：2026-10-21
> 性质：本地授权靶场

---

## 一、目标

在 DVWA 反射型 XSS 模块中，输入一段脚本让浏览器执行，并用 Burp 抓到原始请求、在 Repeater 里改包重放，理解「输入变代码」的完整链路。

## 二、环境

| 项 | 值 |
|---|---|
| 靶场地址 | `http://dvwa.test:8080`（`/etc/hosts` 把 `dvwa.test` 指向 127.0.0.1） |
| 服务器 | `php -S 0.0.0.0:8080`（用 `sudo` 启动） |
| Burp | 监听 `127.0.0.1:8081` |
| 浏览器代理 | 手动代理 `127.0.0.1:8081` |

> 为什么用 `dvwa.test` 而不是 `127.0.0.1`：Firefox 默认**永远不代理 localhost / 127.0.0.1 / ::1**，用假域名才能让请求经过 Burp。

## 三、复现过程

### 3.1 直接打 payload

DVWA 左侧 **XSS (Reflected)**，输入框输入：

```html
<script>alert(1)</script>
```

点 Submit → **弹窗弹出**，URL 变成：

```
http://dvwa.test:8080/vulnerabilities/xss_r/?name=<script>alert(1)</script>#
```

### 3.2 看源码

右下角 View Source → `vulnerabilities/xss_r/source/low.php`：

```php
<?php
header ("X-XSS-Protection: 0");

if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
    echo '<pre>Hello ' . $_GET[ 'name' ] . '</pre>';
}
?>
```

**没有任何转义**，输入原样拼进 HTML，浏览器当代码执行。

### 3.3 Burp 抓包

浏览器提交 payload 后，Burp → Proxy → HTTP history 里出现：

```
GET /vulnerabilities/xss_r/?name=%3Cscript%3Ealert%281%29%3C%2Fscript%3E&Submit=Submit HTTP/1.1
Host: dvwa.test:8080
```

`%3C` = `<`，`%3E` = `>`，`%2F` = `/`，payload 被 URL 编码后发出。

### 3.4 Repeater 改包重放

请求右键 → Send to Repeater，把第一行改成：

```
GET /vulnerabilities/xss_r/?name=test123&Submit=Submit HTTP/1.1
```

点 Send → Response 里（Render 标签）显示：

```html
<pre>Hello test123</pre>
```

**改参数 → 服务器按新参数返回**，证明完全控制了请求。

## 四、原理

- 反射型 XSS：参数在 URL，服务器原样输出到页面，浏览器执行。
- 只影响**点击链接的人**，但可以用短链接、邮件等方式投递。
- `X-XSS-Protection: 0` 还顺手关掉了浏览器自带的过滤器。

## 五、修复

```php
echo '<pre>Hello ' . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8') . '</pre>';
```

| 措施 | 说明 |
|---|---|
| 输出编码 | 核心：`htmlspecialchars` 把 `<` `>` 转成实体 |
| 输入校验 | 限制字符集、长度 |
| CSP | 限制内联脚本执行 |
| 不要关 X-XSS-Protection | 保留浏览器防护 |

## 六、踩坑

1. **Burp 和 DVWA 抢 8080**：DVWA 占 8080，Burp 必须换 8081。
2. **Firefox 不代理 localhost**：设置里那行小字写了 `127.0.0.1 ... are never proxied`，必须用 `dvwa.test` 这类假域名。
3. **Burp 监听要勾 Running**：不勾等于没开，浏览器报 `proxy server is refusing connections`。
4. **No proxy for 不能有 127.0.0.1**：有的话本地请求绕过代理。

## 七、收获

- 反射型 XSS 的本质：**输出未编码**。
- 完整的验证链路：浏览器提交 → Burp 抓包 → Repeater 改包 → 看响应。
- 会读源码，才知道漏洞在哪一行。
