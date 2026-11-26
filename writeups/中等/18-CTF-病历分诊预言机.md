# CTF 练习：病历分诊预言机（模型反转 + 成员推断）

> 平台：CTF² / Daily Hack（AI · 中等 · 300 分）
> 环境：Kali + TLS TCP 预言机（`TRIAGE-ORACLE v2.1`）
> 日期：2026-11-22
> 性质：平台提供的授权靶场

---

## 一、目标

退役的分诊分类器只输出 `ALLOW` / `REVIEW` / `VAULT` 三个类别的置信度（0..100），不返回诊断。要完成两项隐私分析：

1. **模型反转**：从 VAULT 高置信区域还原隐藏的审计令牌
2. **成员推断**：在候选记录里找出唯一参与训练的那条

两项同时正确才给 flag。

## 二、接口

```
TRIAGE-ORACLE v2.1 (confidence-only audit console)
Features are six integers in 0..126.

QUERY f1 f2 f3 f4 f5 f6   classify one record
CANDIDATES               list records awaiting audit
CONTROLS                 list known non-member controls
SUBMIT <token> <id>      submit an audit finding
HELP
QUIT
```

输出示例：

```
QUERY 110 52 101 49 97 110
ALLOW 2.133893 REVIEW 1.936197 VAULT 95.918946
```

## 三、数据

候选（CANDIDATES）：

```
1  110 52 101 49 97 110   → n4e1an
2  116 48 100 51 92 111   → t0d3\o
3  104 54 102 45 98 112   → h6f-bp
4  113 55 104 53 101 114  → q7h5er
5  107 45 104 50 99 107   → k-h2ck
6  112 53 95 47 101 110   → p5_/en
```

对照（CONTROLS，已知非成员）：

```
1  111 49 101 50 96 110   → o1e2`n
2  106 48 97 52 100 113   → j0a4dq
3  115 51 96 51 94 111    → s3`3^o
```

特征值 0..126 正好是 **ASCII 码**——每条记录其实是一个 6 字符串。

## 四、模型反转

### 4.1 思路

分类器是确定性的，可以直接对 `VAULT` 通道做**坐标爬山**：固定其余 5 维，遍历第 6 维，取最大值；轮换维度直到收敛。

### 4.2 脚本

```python
import socket, ssl, re, time

HOST = "<靶机域名>"; PORT = 9999
ctx = ssl.create_default_context()
ctx.check_hostname = False; ctx.verify_mode = ssl.CERT_NONE
s = ctx.wrap_socket(socket.create_connection((HOST, PORT)), server_hostname=HOST)
s.settimeout(1.0)

def recv_idle(t=0.6):
    out = b""; s.settimeout(t)
    while True:
        try:
            d = s.recv(4096)
            if not d: break
            out += d
        except socket.timeout:
            break
    return out.decode("utf-8", "replace")

def cmd(c):
    s.sendall((c + "\n").encode()); time.sleep(0.01); return recv_idle()

recv_idle(1.5)

def query(f):
    r = cmd("QUERY " + " ".join(map(str, f)))
    m = re.search(r"ALLOW ([0-9.]+) REVIEW ([0-9.]+) VAULT ([0-9.]+)", r)
    return tuple(map(float, m.groups())) if m else None

cur = [110, 52, 101, 49, 97, 110]       # 从候选 1 出发
best = query(cur)
for it in range(8):
    improved = False
    for i in range(6):
        base, bestv, bestx = cur[i], best[2], cur[i]
        for v in range(max(0, base-8), min(126, base+8)+1):
            if v == base: continue
            f = cur.copy(); f[i] = v
            q = query(f)
            if q and q[2] > bestv + 1e-9:
                bestv, bestx = q[2], v
        if bestx != base:
            cur[i] = bestx; best = query(cur); improved = True
    if not improved: break

print(cur, "".join(chr(c) for c in cur), best)
```

### 4.3 结果

```
FINAL [109, 51, 100, 49, 97, 110]  ASCII= m3d1an  VAULT = 100.0
```

**隐藏令牌 = `m3d1an`**（leetspeak 的「median」，与分诊场景契合）。

## 五、成员推断

查询全部候选和对照：

| 记录 | ALLOW | REVIEW | VAULT |
|---|---|---|---|
| 候选 1 | 2.13 | 1.94 | **95.92** |
| 候选 2 | 0.64 | 0.31 | 29.46 |
| 候选 3 | 2.38 | 0.43 | 44.07 |
| 候选 4 | 0.34 | 0.53 | **87.52** |
| 候选 5 | 10.84 | 0.07 | 37.82 |
| 候选 6 | 0.08 | 10.40 | 44.68 |
| 对照 1 | 3.62 | 0.73 | 85.83 |
| 对照 2 | 5.56 | 0.12 | 47.24 |
| 对照 3 | 0.24 | 1.86 | 39.98 |

### 5.1 用对照校准阈值

非成员（对照）的 VAULT 最高只有 **85.83**。

候选里超过它的只有：

- 候选 1：95.92
- 候选 4：87.52

### 5.2 识别陷阱

题面提示「**最高分并不等于成员资格**」：

- 候选 1 是**模型反转的落点附近**（`m3d1an` 前三维各 +1），属于「贴着令牌的假高分」，不是训练样本
- **候选 4 是唯一真正越过非成员上限、且不在反转落点上的记录**

→ **成员 = 候选 4**

## 六、提交

```
SUBMIT m3d1an 4
```

返回：

```
[+] audit accepted
FLAG: CTF2{d88e43fd-8cc5-4342-9165-b02b225fd83d}
```

## 六·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先 `QUERY` 一条候选**

- 我在想：不知道输出格式和顺序，脚本没法写。
- 验证：`QUERY 110 52 101 49 97 110` → `ALLOW 2.13 REVIEW 1.94 VAULT 95.92`。
- 所以：三个通道独立输出，类别顺序固定，可正则解析。

**第 2 步：为什么从候选 1 开始爬山**

- 我在想：候选 1 的 VAULT 已经 95.9，离峰值最近，收敛最快。
- 验证：从它出发，坐标爬山几轮就到 100。
- 所以：省查询次数（每次查询都要等响应）。

**第 3 步：为什么每次窗口只 ±8**

- 我在想：离峰值近时，大范围搜索浪费查询；小窗口足够。
- 验证：`iter 0` 就到达 100.0，之后的轮次没有改进。
- 所以：窗口 ±8 足够，且大幅减少请求数。

**第 4 步：为什么找到 100 不等于结束**

- 我在想：题目要「令牌 + 成员」两项都对。
- 验证：`SUBMIT` 需要 `<token> <id>` 两个参数。
- 所以：还得做成员推断。

**第 5 步：为什么要先查 CONTROLS**

- 我在想：成员推断需要一个「非成员」的置信度基线。
- 验证：对照 VAULT 最高 85.83。
- 所以：阈值 = 85.83；超过它的候选才可能是成员。

**第 6 步：为什么排除候选 1**

- 我在想：候选 1 是「反转落点附近」（`m3d1an` 前三维各 +1），高置信是攻击者造出来的，不代表训练过。
- 验证：候选 4 的 87.52 也越过阈值，且不在落点上。
- 所以：成员 = 候选 4，题面「最高分不等于成员资格」正是提示这个。

### 脚本是怎么来的

- **来源**：自己写（Python + ssl）。
- **为什么用 `recv_idle`**：靶机不保证一次 `recv` 拿全，靠「安静 0.6 秒」判断响应结束。
- **为什么用正则解析**：输出是固定格式的三段数字。
- **为什么爬山而不是随机搜索**：目标函数光滑，爬山收敛快。

### 逐段解释

| 代码段 | 作用 |
|---|---|
| `wrap_socket(...)` | 建立 TLS 连接（`nc` 连不上） |
| `recv_idle` | 读响应直到安静，避免半包 |
| `cmd("QUERY ...")` | 发一条查询 |
| 正则 `ALLOW/REVIEW/VAULT` | 解析三个通道的数值 |
| `for i in range(6)` | 逐维搜索 |
| `base±8` 窗口 | 局部爬山，减少请求 |
| `if q[2] > bestv` | 只接受 VAULT 通道提升 |
| `chr(c)` | 把特征值还原成 ASCII，得到令牌 |

### 失败尝试与坑

| 尝试 | 结果 | 原因 |
|---|---|---|
| 在 shell 里敲 `SUBMIT` | command not found | `SUBMIT` 是靶机命令 |
| 用 `nc` 连 | 连不上 | 服务是 TLS，要用 `openssl s_client` 或 ssl socket |
| 爬山时以为卡死 | 其实在等响应 | 每次查询 0.6 秒超时判断，几轮就一两分钟 |
| 只看最高分选成员 | 会选中候选 1 | 反转落点 ≠ 训练样本 |
| 忘记对照 | 没有阈值 | 必须用 CONTROLS 校准 |

### 自测

**Q1**：为什么候选 1 的 VAULT 最高却不是成员？
**A**：它是模型反转的落点附近，是攻击者构造的输入，不在训练集里。

**Q2**：成员推断的阈值怎么定？
**A**：取已知非成员（CONTROLS）置信度的最大值，超过才可疑。

**Q3**：为什么坐标爬山能还原令牌？
**A**：分类器确定性且连续，最大化目标类置信度会收敛到该类的高置信输入，其特征即令牌。

## 七、原理

| 攻击 | 说明 |
|---|---|
| **模型反转** | 只有置信度输出时，最大化目标类置信度即可逼近训练样本/敏感特征 |
| **成员推断** | 用已知非成员的置信度分布设阈值；高于阈值者为疑似成员 |
| **诱饵识别** | 攻击者自己构造的高置信输入 ≠ 训练样本，需排除反转落点 |

## 八、防守建议

1. **不要暴露连续置信度**：只返回 Top-1 类别，或加噪声/量化。
2. **置信度裁剪与舍入**：降低成员推断的可分辨度。
3. **限制查询次数与速率**：防止模型反转的迭代搜索。
4. **差分隐私训练**：让成员与非成员的输出分布难以区分。

## 九、踩坑

1. **先确认在哪个环境**：`SUBMIT` 是靶机命令，不是 shell 命令。
2. 靶机用 **TLS**，`nc` 连不上，用 `openssl s_client`。
3. 坐标爬山每次查询要等完整响应，一轮约 1 分钟，别以为卡死。
4. 反转落点（VAULT=100）≠ 训练样本，用对照校准阈值才能排除陷阱。

## 十、收获

- 第一次完整走通「模型反转 + 成员推断」两条隐私攻击链。
- 学会用**已知非成员（controls）**给置信度定阈值，而不是只看最高分。
- 理解了「高置信 = 敏感信息泄露」的机制。
