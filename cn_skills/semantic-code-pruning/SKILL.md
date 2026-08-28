---
name: semantic-code-pruning
description: "当需要 review 或编辑显得臃肿、冗余、过度防御或 AI 生成痕迹明显的代码时使用，尤其是死赋值、未使用返回值、重复 guard、透传临时变量、薄 wrapper、不可达分支和语义 no-op。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# 裁剪语义 No-Op 代码

## 核心工作流

### 编辑前先理解请求

当用户希望通过删除冗余、死代码、过度防御逻辑或 AI boilerplate 让代码更小的时候，使用这个 skill。

这个 skill 用于保持行为不变的语义级裁剪，不是代码风格整理、formatter、宽泛重构、代码压缩或推测性的架构清理。

### 先阅读项目

在进行裁剪编辑前：

1. 阅读范围内文件，以及能解释运行行为的最近 README、package、build 或框架配置。
2. 搜索所有可能观察到被裁剪代码的 caller、override、implementation、test、fixture、script、生成客户端和公开 export。
3. 追踪相关数据流：赋值到观察点之间是否经过 return、持久化状态、event / log、外部调用、UI 渲染、API 响应或测试断言。
4. 追踪相关控制流：guard、modifier、更早的校验、异常路径、模式匹配、状态机转换和框架生命周期 hook。
5. 检查公开契约是否依赖当前形态：ABI/API 签名、schema、序列化字段、event / log 格式、错误类型、CLI 输出、migration 或生成代码。
6. 找出项目已有的最强验证路径：聚焦测试、类型检查、build、snapshot、trace、gas report 或框架专项检查。

如果无法检查必要的 caller、契约、schema、生成产物或验证路径，需要明确说明，不要把裁剪视为已经完全证明。

### 默认范围

默认只检查用户提到的文件或行为。

如果用户要求做通用裁剪检查，优先检查应用代码或合约源码，再检查相邻测试和公开接口。默认排除 vendored dependencies、生成代码、migration、lockfile、snapshot 和纯格式化 churn，除非用户明确包含它们。

### 裁剪类别

检查以下机会：

- 死赋值和被读取前覆盖的值
- 未使用返回值和未使用输出
- 重复 guard 或已经被强制检查过的条件
- 没有可读性或调试价值的透传临时变量
- 在已证明前置条件或状态转换下不可达的分支
- 没有增加边界、不变量、重试、日志、授权、归一化或领域词汇的薄 wrapper
- 冗余转换、重复计算或 no-op normalization
- AI 生成代码引入的、没有可达目的的过度防御分支

不要机械裁剪。只有当删除后对每个受支持 caller 和输入都保持行为不变时，代码才可以删。

## 裁剪等级

### 1. 安全的直接编辑

当证明局部且完整时，可以直接 patch 低风险裁剪。

常见示例：

- 删除在任何读取前都会被覆盖的赋值
- 内联只赋值一次、读取一次，且名称不承载领域含义的变量
- 当所有到达 caller 都已经强制相同前置条件时，删除不可达 private 分支
- 当所有 caller 都忽略返回值，且没有 interface 要求时，删除 private 返回值
- 删除只调用一个 private helper，且不增加不变量、副作用或有效词汇的 wrapper

只有在没有公开契约、副作用、诊断行为或框架约定依赖当前代码时，才直接 patch。

### 2. 有条件的编辑

只有在行为清晰且验证强度足够时才 patch。

常见示例：

- 合并副作用和错误行为等价的重复分支
- 在所有 mutation 路径都已证明后，删除围绕内部状态的防御性检查
- 简化重复 parsing、serialization 或 normalization 逻辑
- 删除面向已文档化不支持版本或模式的兼容分支
- 删除冗余数值计算，但前提是完全理解 rounding、overflow、执行顺序和精度

只有在满足以下条件时才 patch：

- 不变量来自代码证明，而不是命名推断或当前测试覆盖
- 可观察的 error type、message、log、event、metrics 和求值顺序保持等价
- 不改变公开 API、schema、storage、migration、生成客户端和框架生命周期边界
- 现有测试或检查对受影响行为有足够覆盖

### 3. 只作为建议的编辑

除非用户明确批准更大范围的兼容性或清理改动，否则以下内容保留为建议。

常见示例：

- 改变公开 API、ABI、CLI 输出、序列化字段或数据库 schema
- 删除 migration、兼容 adapter、feature flag、审计日志、telemetry 或 rollback / cleanup 代码
- 删除不可信输入、授权、资产流动、并发、锁、callback 或重入相关校验
- 裁剪生成代码，或必须匹配 schema、反射系统、decorator、dependency injection container、生命周期 hook 的代码
- 删除用于保护文档化配置范围、版本差异或外部集成的未来兼容代码
- 只因为当前测试没用到就删除代码

当证明不完整时，保留代码，并说明还缺什么证据。

## 编辑规则

### 先证明，再 patch

对每个非平凡删除，先证明它为什么不会影响可观察行为。证据可以来自 caller 路径、数据流、控制流、类型约束、状态不变量、接口契约和副作用分析。

### 只 patch 已证明的最小改动

只删除冗余代码。不要顺手重写附近逻辑、重命名符号、重排文件，或修改触达行之外的格式，除非裁剪本身需要。

### 保留可观察边界

把以下内容视为裁剪边界：

- 公开 API、ABI、export、override、schema、序列化输出、CLI 输出、生成客户端
- storage 或数据库 layout、migration 历史、event / log / error 形态、metrics、tracing、审计轨迹
- 输入校验、访问控制、auth / session 检查、资产流动、锁、并发、cleanup、rollback
- 框架生命周期 hook、反射、decorator、dependency injection、dynamic import、生成代码
- 承载业务含义、会计分类、协议状态或团队约定的领域命名

如果代码看起来啰嗦但保护了这些边界，保留它，或者建议用更清晰的注释替代删除。

### 遇到歧义就停止

如果删除可能以非平凡方式改变行为、兼容性、诊断信息、执行顺序、持久化或可观测性，不要猜。指出可疑代码，解释风险，并保留为建议。

### 保持范围收敛

不要把裁剪检查变成宽泛重构、安全审计、优化专项、格式化清理或测试重写。用户明确要求时，再把这些工作单独路由。

## 领域边界

对于 Solidity 和 Foundry 项目，除非用户明确要求更大的兼容性改动，否则保留 external / public 函数签名、event 形态、custom error selector、storage 变量顺序、initializer 参数、modifier、访问控制、upgradeable storage layout 和 revert 行为。对于外部输入、资产流动、oracle / pair 假设、权限检查、callback 和重入边界，只有在所有可达入口都证明不变量成立时才删除相关 guard。

对于 TypeScript、JavaScript、Go、Python 等应用代码，要特别小心 reflection、serialization、decorator、dynamic import、框架生命周期函数、dependency injection、生成类型、公开 export，以及可能被外部消费的 logging 或 telemetry。

## 验证

优先使用项目已有的最强验证路径。

优先级：

1. 受影响行为的聚焦测试或回归测试
2. 能验证触达契约的类型检查、lint 或框架检查
3. build、snapshot、生成客户端检查、schema 检查或 trace 对比
4. 当触达代码是共享逻辑或外部可见行为时，运行更宽的测试套件

对于 Solidity 和 Foundry 项目，优先使用触达文件的 `forge fmt --check`、`forge build` 和受影响流程的聚焦 `forge test`。

如果验证失败，需要说明失败看起来是由以下哪类原因导致：

- 裁剪改动本身
- 仓库中已有的问题
- 与裁剪 patch 无关的项目配置或依赖问题

如果没有报告验证结果，不要声称裁剪工作已经完成。

## 输出约定

默认最终输出应包含：

- 已 review 的文件和相关 caller
- 裁剪范围
- 已实现的删除或简化
- 每个非平凡删除保持行为不变的证明
- 有意保留的可疑代码
- 风险说明
- 验证结果

裁剪总结需要分为：

- removed semantic no-ops
- preserved boundaries
- deferred recommendations

对于跳过的项目，需要包含原因：

- 证明不完整
- 公开契约风险
- 安全或运维边界
- 生成代码或框架拥有的代码
- 兼容性或 migration 风险
- 验证信心不足

## 响应风格

- 直接
- 基于具体文件
- 优先追求保持行为不变的删除，而不是聪明压缩
- 不要只用测试作为冗余证明
- 不要模糊裁剪、优化、安全审计和重构之间的边界
