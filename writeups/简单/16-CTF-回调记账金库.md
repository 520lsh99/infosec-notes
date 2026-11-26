# CTF 练习：回调记账金库（CallbackVM 重入式双倍记账）

> 平台：CTF² / Daily Hack（Blockchain · 简单 · 100 分）
> 环境：离线 CallbackVM 伪汇编 + JSON 初始状态（无节点/无网络）
> 日期：2026-11-21
> 性质：平台提供的授权靶场

---

## 一、目标

给一份 CallbackVM 伪汇编和初始状态，审计「带重入锁的信用金库」和「转账时回调接收方」的代币合约，算出固定脚本执行 11 轮后**金库与攻击者账户的最终 RWD 余额**。

提交格式：

```
flag{<vault_final_RWD_balance><attacker_final_RWD_balance>}
```

每个余额 **16 位小写十六进制、补零、无分隔符**。

## 二、初始状态

```json
"token_ledger_RWD": {
  "0xa77ac": 1234,   // 攻击者（caller）
  "0x7a1117": 4096,  // 金库（Vault，hook 注册地址）
  "0x40a57": 5000    // 诚实账户
},
"vault_credit": { "0xa77ac": 0, "0x40a57": 4096 },
"vault_lock": 0
```

脚本：

```
repeat 11:
  deposit(137)
  withdrawAll()
```

## 三、关键代码

### 3.1 deposit（漏洞所在）

```
Vault.deposit:
    TOKEN_XFER caller, self, [arg0]   ; 转账给金库，触发 onRWDReceived
    LOADM caller
    PUSH [arg0]
    ADD
    STOREM caller                     ; 又给 caller 记一次
```

### 3.2 回调

```
Vault.onRWDReceived:
    LOADM [arg0]
    PUSH [arg1]
    ADD
    STOREM [arg0]     ; credit[from] += amt
```

`TOKEN_XFER` 的语义：转账完成后，如果 `hook[to]` 存在，先调用回调再返回。

因为 `to = self = 0x7a1117`，而 `hook[0x7a1117] = Vault.onRWDReceived`，**回调必然触发**。

### 3.3 withdrawAll（锁没防住问题）

```
LOAD 2            ; locked?
JZ unlock
REVERT "reentrancy lock"
unlock:
PUSH 1 STORE 2    ; 加锁
LOADM caller STORE 3
PUSH 0 STOREM caller
TOKEN_XFER self, caller, [S3]
PUSH 0 STORE 2    ; 解锁
```

锁只防 `withdrawAll` 重入，**但这个漏洞根本不在重入，而在 deposit 的重复记账**。

## 四、漏洞分析

一次 `deposit(137)`：

1. `TOKEN_XFER caller → vault 137` 成功
2. 触发 hook → `credit[caller] += 137` ← **第一次**
3. deposit 继续 → `credit[caller] += 137` ← **第二次**

所以每轮：

```
credit[caller] = 274
```

接着 `withdrawAll()` 把 274 全额提走。

## 五、逐轮模拟

| 轮 | deposit 137 | credit | withdrawAll 274 | 攻击者 RWD | 金库 RWD |
|---|---|---|---|---|---|
| 0 | — | 0 | — | 1234 | 4096 |
| 1 | −137 | 274 | +274 | 1371 | 3959 |
| 2 | −137 | 274 | +274 | 1508 | 3822 |
| … | … | … | … | +137/轮 | −137/轮 |
| 11 | — | — | — | **2741** | **2589** |

计算结果：

```
攻击者 = 1234 + 11 × 137 = 2741 = 0xAB5
金库   = 4096 − 11 × 137 = 2589 = 0xA1D
```

守恒校验：`2741 + 2589 + 5000 = 10330`，与初始总额一致 ✓

## 六、提交

```
flag{0000000000000a1d0000000000000ab5}
```

- 金库：`2589` → 16 位十六进制 `0000000000000a1d`
- 攻击者：`2741` → 16 位十六进制 `0000000000000ab5`

## 七、修复建议

| 问题 | 修复 |
|---|---|
| 回调和调用方重复记账 | **先记账后转账**（Checks-Effects-Interactions） |
| 依赖回调修改状态 | 转账回调不应改变调用方的会计状态 |
| 重入锁形同虚设 | 锁要覆盖「状态修改 + 外部调用」整个流程 |
| 缺少对账 | 定期校验 `sum(RWD ledger) == 总量` |

正确写法：

```
Vault.deposit(amt):
    credit[caller] += amt      # 先记账
    TOKEN_XFER caller, self, amt   # 后转账
```

或者回调里不记账，只做通知。

## 八、踩坑

1. **地址映射容易搞混**：`0xa77ac` 是 caller，`0x7a1117` 才是金库（hook 注册地址）。
2. **hook 触发条件**：只有 `hook[to]` 存在才回调；`to = self`。
3. **锁的位置**：以为要绕锁，其实漏洞在 deposit，与锁无关。
4. **余额定义**：README 要的是 **RWD 账本余额**，不是 `vault_credit`。

## 九、收获

- 认清了「回调式重入」的变体：不是同一函数重入，而是**回调与主流程各记一次账**。
- 学会用守恒律（总额不变）校验模拟结果。
- 理解了 Checks-Effects-Interactions 为什么是智能合约的第一条铁律。
