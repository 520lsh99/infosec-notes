# CTF 练习：vaultbridge —— JNI 边界封印桥（JNI 引用与线程约束）

> 平台：CTF² / Daily Hack（Mobile · 困难 · 500 分）
> 环境：Kali + 明文/TLS TCP 菜单（openssl s_client）
> 日期：2026-11-20
> 性质：平台提供的授权靶场

---

## 一、目标

模拟 `libvaultbridge.so` 的 JNI 边界，菜单里可见 RegisterNatives 方法表、类对象身份、局部/全局引用和 native 工作线程。要在 **native 工作线程**上完成 `unseal`，取回 flag。

## 二、菜单

```
1) classes        6) DeleteRef
2) FindClass      7) AttachCurrentThread
3) InvokeNative   8) DetachCurrentThread
4) refs           9) ExceptionClear
5) NewGlobalRef   10) unseal
0) help           q) quit
```

```
3 InvokeNative <class> <method> <sig> [arg ...]
    args: s:<hex>  byte array | i:<int> integer | ref:<handle> reference
10 unseal <seal-ref> <class-ref>  on the native worker
```

每次操作后回显状态行：

```
[state] thread=main tid=0x1a2b | worker=none | pending=none | locals(main)=0/16
```

## 三、三个 JNI 约束（本题考点）

### 3.1 局部引用不能跨线程

`2 FindClass` 在主线程得到的 `L1` 只在主线程有效：

```
L1 local tid=0x1a2b kind=class identity=0x5eed0001
```

### 3.2 类对象的「发现路径」决定身份

- 主线程 / `JNI_OnLoad` 拿到：`identity=0x5eed0001`（App ClassLoader）
- `AttachCurrentThread` 后的 native 线程再 FindClass：`identity=0x5eed0002`（系统 ClassLoader）

**Android 经典坑**：native 线程 FindClass 走系统 ClassLoader，拿不到 App 类。
**正确做法**：在主线程拿到类 → `NewGlobalRef` 提升为全局引用，供工作线程使用。

### 3.3 挂起异常会阻断后续 JNI 调用

把引用名当类名传给 InvokeNative 会触发：

```
ClassNotFoundException: L1
pending=java.lang.ClassNotFoundException
```

之后所有 JNI 调用被挡，必须先 `9 ExceptionClear`。

## 四、复现步骤

```
2                                    # FindClass
com/example/native_vault/NativeVault  → L1 (identity 0x5eed0001, main)

5                                    # NewGlobalRef
L1                                   → G1 (global, identity 0x5eed0001)

7                                    # AttachCurrentThread → worker

3                                    # 在 worker 上生成 seal
com/example/native_vault/NativeVault guard ([BILjava/lang/Class;)V s:00 i:0 ref:G1
                                     → L2 (seal, tid=0x1a2c)

5                                    # seal 提升为全局
L2                                   → G2 (kind=seal)

10                                   # 解封
G2 G1
                                     → JNI: unseal -> OK
                                     → CTF2{...}
```

## 五、关键细节

| 细节 | 说明 |
|---|---|
| `guard` 有两个重载 | `(I)I` 只是状态查询；`([BILjava/lang/Class;)V` 才生成 seal |
| 签名容易看错 | 是 `([BILjava/lang/Class;)V`（byte 数组 + int + Class），不是 `([Ljava/lang/Class;)V` |
| seal 绑定创建线程 | `origin_tid` 必须是 worker，主线程生成的 seal 在 worker 上会报 `seal is bound to tid ...` |
| 两个参数都必须是全局引用 | 局部引用跨回调无效 |
| `unseal` 不能用 3 调 | 只能通过 `10` 执行，且必须没有挂起异常 |

## 六、错误分类（题目的提示）

| 报错 | 原因 |
|---|---|
| `NoSuchMethodError` | 签名写错（如漏了 `[BI`） |
| `ClassNotFoundException: L1` | 把引用名当类名 → 挂起异常 |
| `local ref ... invalid` | 跨线程使用局部引用 |
| `seal is bound to tid ...` | seal 不在目标线程创建 |
| `callback parameters must be global` | 传了局部引用 |

## 六·五、复盘四问

### 推理链（为什么这么做）

**第 1 步：为什么先看 `classes`**

- 我在想：不知道有哪些 native 方法、签名是什么，就没法调用。
- 验证：`classes` 列出 `guard (I)I`、`guard ([BILjava/lang/Class;)V`、`unseal (Ljava/lang/Class;)V`，以及 dlsym 导出 `unseal [retired]`。
- 所以：**只信 RegisterNatives 表**；dlsym 那个是干扰项。

**第 2 步：为什么在主线程 FindClass**

- 我在想：FindClass 与线程绑定，放 worker 上可能拿到不同的类对象。
- 验证：主线程 `FindClass` → `L1 (identity=0x5eed0001)`；worker 上再 `FindClass` → `L2 (identity=0x5eed0002)`。
- 所以：**App 类只能在主线程/JNI_OnLoad 路径拿到**（Android 经典坑）。

**第 3 步：为什么立刻 NewGlobalRef**

- 我在想：`L1` 是主线程局部引用，切到 worker 就失效。
- 验证：直接拿 `L1` 去 worker 调用会报局部引用无效。
- 所以：`5 → L1 → G1`，全局引用跨线程有效。

**第 4 步：为什么 `unseal` 用 `3` 调不动**

- 我在想：`3` 只能调 RegisterNatives 里的方法。
- 验证：`3 ... unseal (Ljava/lang/Class;)V` 报 `NoSuchMethodError`。
- 所以：`unseal` 只能走 `10`，且必须在 worker 上、没有挂起异常。

**第 5 步：为什么 seal 必须在 worker 上生成**

- 我在想：seal 对象记录 `origin_tid`，跨线程使用会校验失败。
- 验证：主线程生成的 seal 在 worker 上用，报 `seal is bound to tid ...`。
- 所以：`7` 先 attach，再 `3` 调 `guard ([BILjava/lang/Class;)V` 生成 `L2 (seal)`。

**第 6 步：为什么两个参数都要全局引用**

- 我在想：`10` 的 usage 明确写 `<seal-global-ref> <class-global-ref>`。
- 验证：传 `G1` 当 seal → `not a seal handle`；传局部引用 → `callback parameters must be global`。
- 所以：`5 → L2 → G2`，最后 `10 G2 G1`。

**第 7 步：为什么 `8` 不能用**

- 实测：`8 DetachCurrentThread` 之后会话直接断开（`unexpected eof`）。
- 所以：整个流程避开 `8`。

### 命令/载荷是怎么来的

- **来源**：全部手输菜单命令，无脚本。
- **签名 `([BILjava/lang/Class;)V` 怎么来的**：从 `classes` 输出抄，**注意包含 `[BI`**（byte 数组 + int + Class），少抄就 `NoSuchMethodError`。
- **类名**用斜杠：`com/example/native_vault/NativeVault`。

### 逐段解释

| 命令 | 作用 |
|---|---|
| `2` + 类名 | 主线程 FindClass，得到局部引用 `L1`（identity 01） |
| `5` + `L1` | 提升为全局 `G1`，供 worker 使用 |
| `7` | AttachCurrentThread，切到 native worker |
| `3` + `guard ([BILjava/lang/Class;)V s:00 i:0 ref:G1` | 在 worker 上生成 seal 局部引用 `L2` |
| `5` + `L2` | seal 提升为全局 `G2`（kind=seal） |
| `10` + `G2 G1` | 在 worker 上解封，输出 flag |

### 失败尝试与坑

| 尝试 | 报错 | 原因 |
|---|---|---|
| 签名写成 `([Ljava/lang/Class;)V` | `NoSuchMethodError` | 漏了 `[BI`（本机最难发现的一个坑） |
| 类名用点号 | `ClassNotFoundException` | FindClass 要斜杠 |
| 用 `L1` 跨线程 | 局部引用无效 | 局部引用绑线程 |
| 主线程生成 seal 再 worker 用 | `seal is bound to tid` | seal 绑定创建线程 |
| 执行 `8 DetachCurrentThread` | 会话断开 | 该命令会回收会话 |
| FindClass 失败后继续操作 | 全部被挡 | 挂起异常未清除，要先 `9` |

### 自测

**Q1**：为什么主线程和 worker 上 FindClass 得到不同 identity？
**A**：worker 线程走系统 ClassLoader，拿不到 App ClassLoader 加载的类。

**Q2**：`unseal` 为什么不能用 `3` 调？
**A**：它不在 InvokeNative 的可调表里，只能通过命令 `10` 在 worker 上执行。

**Q3**：seal 为什么必须在 worker 上生成？
**A**：seal 记录 `origin_tid`，使用时会校验创建线程。

## 七、修复与启示

这是逆向/安全研究中的「JNI 调试 harness」，真实开发中的对应规则：

1. **native 线程不要 FindClass**，改用 JNI_OnLoad 缓存 + 全局引用。
2. **局部引用只在本线程、本帧有效**，跨线程/回调必须 NewGlobalRef。
3. **异常挂起时不要继续调 JNI**，先 ExceptionCheck / ExceptionClear。
4. **线程绑定对象**（如 seal）必须在目标线程创建。

## 八、踩坑

1. **签名看错**：`([Ljava/lang/Class;)V` vs `([BILjava/lang/Class;)V`，卡了很久。
2. **`8 DetachCurrentThread` 会让会话断开**，别用。
3. **FindClass 失败会留下 pending 异常**，不 `9` 清除后面全被挡。
4. 类名用**斜杠**（`com/example/...`），点号会 ClassNotFound。
5. `unseal` 不能用 `3` 调用，只能 `10`。

## 九、收获

- 完整走了一遍 JNI 的四个约束：线程、引用、ClassLoader、异常。
- 理解了「类对象身份不同」在 Android 上意味着什么。
- 学会用菜单回显的错误信息做**错误分类定位**，而不是盲试。
