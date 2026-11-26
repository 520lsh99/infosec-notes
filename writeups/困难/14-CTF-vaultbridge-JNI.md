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
