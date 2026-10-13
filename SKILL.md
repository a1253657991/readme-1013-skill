---
name: readme-1013
description: Task instructions from README1013.md. Use when the user asks to run or follow the README1013 task.
---

# 任鸟飞逆向课程 · AppleProtocolReplica 授权靶场

> ⚠️ **重要提示**  
> 本靶场是一个**模拟分析环境**，其中包含的软件、配置和代码片段来自真实项目分析过程，但**所有结论均基于静态观察和推理，尚未经过完整动态验证**。  
> **本仓库可能包含有意设计的误导线索、反调试手段和混淆特征**，旨在考察分析者的判断力和证据链构建能力。  
> **请勿将本仓库中的任何信息直接当作最终事实**，所有分析均需结合实际动态取证和交叉验证。  
> 仅供授权安全研究与技术学习使用，严禁用于非法目的。

---

## 🎯 靶场核心目标

**本靶场的最终目标是：**

1. **完整理解整个工作流程**：从 .NET 8 托管程序集 → DNGuard HVM JIT 保护层 → RuikeyzSDK 授权中台 → Apple Store HTTP 自动化购买链路（FulfillmentQuery → BuyabilityQuery → AddToBag → Checkout → Shipping → Payment → Review → SubmitOrder）→ 直到最终在苹果官网完成抢购的全链路行为。
2. **还原被保护的核心源码**：通过静态反编译（ILSpy/dnSpy）、CLI 元数据解析、字符串堆分析等手段，理解 `AppleProtocolReplica.dll` 中关键算法、数据结构和协议逻辑，达到**可复现、可解释**的源码级理解。

**换句话说：你不是在"用"这个工具，而是在"拆解"它，理解它如何在 DNGuard 保护下仍然能被静态分析，以及 RuikeyzSDK 如何实现云端授权控制。**

---

## 🎖️ 将军令·必答题（满分要求）

> ⚔️ 以下两项任务由将军亲自部署，**工作量大、难度高，必须认真完成并交出满分答卷**。
> README 原文未覆盖此步骤，但将军令强制要求完成，不得以静态推测代替动态取证交差。

### 将军令任务一：逆运行时本身的原生解密例程（前置必做）

**背景**：DNGuard HVM 的机制是——`HVMRuntm.dll` 被加载后，通过 `VMRuntime` 导出函数拦截 CLR JIT 请求，将被保护方法的真实 IL 字节码在运行时动态解密并注入回内存。要理解托管层 stub 行为，必须先理解原生解密例程本身，否则托管层分析建立在黑箋之上。

**分析要求**：
- [ ] 定位 `HVMRuntm.dll` 中所有执行字节解密的原生函数，给出各函数 RVA 和算法类型
- [ ] 追踪解密例程完整调用链：`VMRuntime` 入口 → 密文缓冲区定位 → 解密执行 → IL 注入回写，画出调用图
- [ ] 确认解密触发条件：逐次 JIT 触发还是首次后缓存？缓存结构在哪里？
- [ ] 确认解密后 IL 写入目标：回写到 MethodDef.RVA 所指内存，还是独立缓冲区？
- [ ] **产出**：解密函数清单（RVA、大小、算法） + 调用图 + 触发机制说明

### 将军令任务二：在 REA 中对 HVMRuntm.dll 做完整原生逆向，找 VMRuntime 初始化与密鑰派生

**背景**：`HVMRuntm.dll`（i386，1,818,624 字节）是 32 位原生 DLL，由 FieldRVA blob 嵌入 `AppleProtocolReplica.dll`，运行时由 `ZYXDNGuarder.a()` 写出并加载。其 `VMRuntime` 导出是 DNGuard HVM 与 CLR JIT 挟钉的核心入口。

**分析要求**：
- [ ] 在 REA 中打开 `HVMRuntm.dll`，使用 `binary_overview` 确认架构（i386）、导出表、导入表
- [ ] 定位 `VMRuntime` 导出函数入口 RVA，使用 `procedure_pseudo_code` 生成伪代码，理解其参数语义
- [ ] 追踪内部初始化子例程：运行时状态结构体初始化、全局密鑰表加载、JIT hook 注册方式
- [ ] 提取密鑰派生完整流程：种子数据来源、KDF 算法类型、最终密鑰存储位置
- [ ] 检查是否存在反分析保护点，列出各保护点的 RVA
- [ ] **产出**：VMRuntime 函数伪代码 + 初始化调用图 + 密鑰派生逆向报告 + 反分析保护点清单

---

## 📚 学习路径（建议顺序）

| 阶段 | 目标 | 产出 |
|------|------|------|
| **阶段零-A：运行时原生解密例程逆向** ⚔️ | 在进入托管层分析之前，必须先逆向 `HVMRuntm.dll` 本身的原生解密例程——这是理解 JIT stub 被动态还原的前提，README 不要求此步但将军令强制要求 | 解密函数地址清单、算法伪代码 |
| **阶段零-B：VMRuntime 初始化与密钥派生** ⚔️ | 使用 REA 对 `HVMRuntm.dll`（32 位，1,818,624 字节）做完整原生逆向，定位 `VMRuntime` 导出函数的初始化流程，提取密钥派生算法（KDF 参数、种子来源、派生路径）| 初始化调用图、KDF 逆向报告 |
| **阶段一：外壳识别** | 区分 .NET 托管层、DNGuard HVM 保护层、RuikeyzSDK 授权层的职责边界 | 画出架构图 |
| **阶段二：静态反编译** | 使用 ILSpy / dnSpy / ilspycmd 反编译 `AppleProtocolReplica.dll`，识别 stub 方法与真实方法 | 反编译伪代码、stub 分布统计 |
| **阶段三：元数据解析** | 手工解析 CLI metadata（`#-` 未压缩流 + `MethodPtr` 间接表），提取 FieldRVA 嵌入 blob | 类型列表 JSON、嵌入 DLL 识别 |
| **阶段四：协议还原** | 从 RuikeyzSDK 请求 Model 和 Apple Store 表单模板，还原完整通信协议 | 请求结构、字段语义、签名算法 |
| **阶段五：授权机制理解** | 分析 AllowLiveRequests 门控、AppleWorkflowMode 三种模式（OfflineFixture / LocalFullFixture / LiveParameterized）| 授权边界说明文档 |
| **阶段六：安全评估** | 判断哪些行为存在风险，哪些属于误导，哪些是真实攻击面 | 安全分析报告 |

---

## 🧩 你需要回答的核心问题

- `AppleProtocolReplica.dll` 的 DNGuard HVM 是如何工作的？为什么 93.1% 的方法体是 `throw` stub，但程序仍然能运行？
- `HVMRuntm.dll`（i386, 1,818,624 字节）和 `HVMRun64.dll`（amd64, 3,784,192 字节）是如何被还原并加载的？触发条件是什么？
- Apple Store 购买链路的 11 步表单是如何串联的？`FulfillmentSelection` 控制了哪些分支？
- Apple Shield（`shld`）挑战的通过方式是 cookie 还是 token？`PythonStoreChallenge.SolvePow / SolveProduct` 各自负责什么？
- `wininet!InternetGetCookieEx / SetCookieEx` 的作用是什么？它在浏览器模式和 HttpClient 模式之间做了什么？
- RuikeyzSDK 收集了哪些设备指纹字段？`maccode` 是如何生成的（ProcessorId + MachineGuid）？
- 为什么即使没有授权也能进行一定程度的操作？`AllowLiveRequests = false` 时程序的行为边界在哪里？
- NTP 时间同步（`NtpTimeSource`、`PropagationDelay`）在抢购中扮演什么角色？精度要求是多少？

---

## 🧪 挑战任务（按难度分级）

### 入门级
- 使用 PE-bear 查看 `AppleProtocolReplica.dll` 的 CLI header，找到 metadata root 和各 stream 的偏移
- 使用 FLOSS 提取字符串，找出所有可疑的 Apple API URL、RuikeyzSDK 域名和 DNGuard 标志字符串
- 统计 `#US`（用户字符串堆）的大小，解释为何只有 733 字节却有 17,659 个方法

### 进阶级
- 使用 ilspycmd / ILSpy 反编译 `AppleProtocolReplica.dll`，找到 `ZYXDNGuarder.Startup()` 和 `ZYXDNGuarder.a()` 方法，理解 HVM runtime 的加载逻辑
- 在 PE 文件中定位 `FieldRVA` 表，找到两个嵌入 DLL blob 的偏移和大小，验证其 PE 头部是否合法
- 列出 RuikeyzSDK 的所有 25 个请求 Model，整理 `In_DataBaseArgs` 基类的三个固定字段及其语义
- 从 `AppleProtocolOptions.cs` 提取完整的 11 步购买表单模板字符串，理解每一步的 HTTP 请求结构

### 高手级
- 编写 Frida 脚本，在 Windows 环境中动态 Hook `VMRuntime` 导出函数，记录 DNGuard JIT 还原方法体时的实际 IL 字节码
- 分析 `AppleHashcash` 的 PoW 算法实现，理解其与 Apple SRP-6a 登录流程的关系
- 解析 `PickupStoreCatalog` 嵌入资源（ManifestResource，92,593 字节），验证 324 家门店数据的完整性（269 US / 49 CN / 6 HK）
- 完整还原 RuikeyzSDK 心跳协议：理解 `heartbeatkey` 的滚动机制和 `surpluspointvalue` 的扣减逻辑

---

## 🔍 静态分析关键发现（供参考，需动态验证）

以下内容基于静态观察，**不构成最终结论**，**可能包含误导线索**。

### 能力摘要（来自导入表 / P/Invoke）

| 能力类别 | 相关 API | 是否实际调用？ |
|----------|----------|----------------|
| HVM 保护运行时 | `HVMRuntm!VMRuntime`（i386）、`HVMRun64!VMRuntime`（amd64）| ✅ 静态确认（FieldRVA 嵌入） |
| Cookie 桥接 | `wininet!InternetGetCookieEx`、`InternetSetCookieEx` | ✅ 静态确认（P/Invoke）|
| 文件操作 | `kernel32!CreateFile`、`DeviceIoControl`、`CloseHandle` | ✅ 静态确认（P/Invoke）|
| 动态加载 | `kernel32!LoadLibrary`、`lstrcpyn` | ✅ 静态确认（P/Invoke）|
| 网络通信 | `HttpClient` → `api.ruikeyz.com`、Apple Store API | ❓ 待动态验证 |
| SRP-6a 登录 | `AppleSrpClient`、PBKDF2、ModPow | ❓ 待动态验证 |
| HTML 解析 | `AngleSharp.dll` | ✅ 静态确认（程序集依赖）|
| NTP 时钟同步 | `NtpTimeSource`、`ServerScheduleClock` | ❓ 待动态验证 |

### 授权/心跳相关字段（RuikeyzSDK 字符串片段）

```
ProcessorId          ← maccode 的组成部分（+ MachineGuid）
platformusercode     ← 卡密账号
inisoftkey           ← 软件唯一 Key
goodscode            ← 商品码
sign                 ← 请求级签名
timestamp            ← 防重放时间戳
platformtypeid       ← 平台类型
requestflag          ← 请求标识
heartbeatkey         ← 心跳滚动 Key
surpluspointvalue    ← 剩余积分（服务端下发）
endtime              ← 授权到期时间
servertimestamp      ← 服务端时间戳
softInfo / softname  ← 远程配置变量（getRemoteVar）
```

> 注意：这些字段已在 RuikeyzSDK 源码层面静态确认，不等于实际请求已发送。

### 配置参数片段（AppleProtocolOptions）

```csharp
// 购买链路 11 步模板
FulfillmentQuery      → BuyabilityQuery    → AddToBagBody
→ CheckoutBody        → UpdateQuantityBody → DeleteBody
→ ShippingBody        → PaymentBody        → ReviewBody
→ SubmitOrderBody     → OrderStatusBody

// 执行模式
AppleStoreExecutionMode { HttpClient, Browser }

// 安全门控
bool AllowLiveRequests   // false = 离线/本地模式，true = 真实 Apple 请求
AppleWorkflowMode { OfflineFixture, LocalFullFixture, LiveParameterized }
```

> 这些参数已由 ilspycmd 反编译静态确认，但实际运行时的赋值需动态验证。

---

## ⚠️ 分析纪律要求

- **不因 `.NET 托管程序集` 先入为主认为没有保护**——DNGuard HVM 是 JIT-hook 保护，静态 RVA 有效但方法体是 decoy stub
- **不把 MethodDef.RVA≠0 等同于"方法未被保护"**——这是 JIT-hook 保护的核心特征，RVA 始终非零
- **不把静态字符串直接等同于动态行为**——`#US` 堆只有 733 字节，运行时字符串由 HVM runtime 解密提供
- **不因发现 stub 方法就放弃分析**——`ZYXDNGuarder`、枚举类、接口类（共 48 个文件）的方法体是真实可读的
- **所有动态测试须在隔离虚拟机（Windows）中进行**
- **动态结果须明确标注：动态确认 / 动态未命中 / 动态受阻**

---

## 📌 动态验证建议工具链

- **REA**：原生 DLL 逆向（`HVMRuntm.dll` 将军令任务主力工具），使用 `binary_overview`、`procedure_pseudo_code`、`get_call_graph`、`list_procedures` 等
- **dnSpy**：.NET 托管程序集调试，可在 JIT 后捕获还原的方法体
- **x64dbg**：原生层 Hook，监控 `HVMRun64!VMRuntime` 调用
- **Frida**：只读观测 API 调用（`LoadLibrary`、`InternetGetCookieEx` 等）
- **Wireshark / Fiddler**：记录 `api.ruikeyz.com` 授权请求和 Apple Store HTTP 流量
- **Procmon**：监控 HVM runtime DLL 的临时文件写出行为
- **ILSpy / ilspycmd**：静态反编译，识别 stub 分布和可读类型

---

## 🧠 适用人群

- .NET 逆向工程进阶学习者（需具备 CLI metadata 基础）
- 对 .NET Protector 分析感兴趣的安全研究者
- 想深入理解"商业 SaaS 授权中台"与"运行时保护"组合架构的安全从业人员

---

## ⚠️ 免责声明

- 本靶场仅用于授权教育与技术研究。
- 请勿将分析所得技术用于破解或绕过软件授权保护。
- 请勿将分析所得技术用于攻击他人计算机系统。
- 作者与课程方不对任何滥用行为负责。

---

## 📧 联系与反馈

如有问题或建议，欢迎提交 Issue 或联系课程助教。

---

> **保持分析纪律，警惕 stub 陷阱，以动态证据为准绳，最终目标——理解 DNGuard HVM 的保护边界和 RuikeyzSDK 的授权机制，而不是依赖反编译结果的表面。**
