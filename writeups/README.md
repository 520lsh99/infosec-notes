# Writeup 索引

> 按难度分目录：`简单/` `中等/` `困难/`。
> 每篇按「推理链 → 代码来源 → 逐段解释 → 失败尝试 → 自测」组织，模板见 [_模板.md](_模板.md)。

## 简单（15 篇）

- [DVWA SQL 注入（Low）](简单/01-DVWA-SQL注入-Low.md) — 布尔/联合注入，从确认到拖库
- [DVWA XSS 反射型（Low）](简单/02-DVWA-XSS-反射型.md) — 弹窗到 Burp 抓包
- [DVWA XSS 存储型（Low）](简单/03-DVWA-XSS-存储型.md) — 留言板种脚本
- [DVWA 文件上传（Low）](简单/04-DVWA-文件上传-Low.md) — 20 字节拿到命令执行
- [极客大挑战 2019 Http](简单/05-CTF-极客大挑战2019-Http.md) — 伪造请求头绕过三层校验
- [源码地图泄露](简单/06-CTF-源码地图泄露.md) — Source Map 泄露源码
- [HTTP 流量分析](简单/08-CTF-HTTP流量分析.md) — webshell 取证
- [HCTF 2018 WarmUp](简单/10-CTF-HCTF2018-WarmUp.md) — 文件包含 + 白名单绕过
- [极客大挑战 2019 EasySQL](简单/11-CTF-极客大挑战2019-EasySQL.md) — 万能密码
- [ACTF2020 Include](简单/13-CTF-ACTF2020-Include.md) — php://filter 读源码
- [ACTF2020 Exec](简单/15-CTF-ACTF2020-Exec.md) — 命令注入
- [回调记账金库](简单/16-CTF-回调记账金库.md) — CallbackVM 回调双倍记账
- [极客大挑战 2019 Havefun](简单/17-CTF-Havefun.md) — 源码注释 + GET 参数
- [极客大挑战 2019 LoveSQL](简单/19-CTF-LoveSQL.md) — 联合查询注入七步法
- [谷仓批号作业台](简单/23-CTF-谷仓批号作业台.md) — JS 逆向算 32 位批号

## 中等（5 篇）

- [链上金库 · 零头审计](中等/07-CTF-链上金库-零头审计.md) — 取整方向缺陷
- [poison-null-frame](中等/09-CTF-poison-null-frame.md) — 帧指针毒化与栈迁移
- [残留驻留（内存取证）](中等/12-CTF-残留驻留-内存取证.md) — 进程对账 + 页表翻译 + 异或
- [病历分诊预言机](中等/18-CTF-病历分诊预言机.md) — 模型反转 + 成员推断
- [ParcelPost 二次渲染](中等/21-CTF-ParcelPost-二次渲染.md) — 作用域逃逸

## 困难（3 篇）

- [vaultbridge · JNI 边界封印桥](困难/14-CTF-vaultbridge-JNI.md) — JNI 引用/线程/ClassLoader/异常
- [Voltcraft VC-2 功耗侧信道](困难/20-CTF-Voltcraft-侧信道.md) — CPA + 掩码复用
- [第九门 · 补码凭条](困难/22-CTF-第九门-补码凭条.md) — 有符号/无符号混淆越界写

---

总计 23 篇。新增篇目请按难度放入对应目录，并更新本索引。
