---
name: foundry-post-dev-optimization
description: "当 Solidity 或 Foundry 合约开发已经完成，并且需要进行开发后的优化检查时使用。优化范围包括 Gas、语义 no-op 裁剪、代码结构和可维护性，但不要把它扩展成安全审计或完整重构。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# Solidity 合约开发后的优化

## 核心工作流

### 编辑前先理解请求

当 Solidity / Foundry 开发已经完成，并且用户希望对现有合约进行一次聚焦的优化检查时，使用这个 skill。

这个 skill 同时覆盖开发后的 Gas 优化和保持行为不变的语义级裁剪。冗余变量、死分支、薄 wrapper 和重复计算，如果删除后能改善 Gas、代码结构或可维护性，都属于优化工作的一部分。

这个 skill 不适用于初始功能实现、风格清理、宽泛重构、代码压缩、安全审计或推测性的架构清理。

### 先阅读项目

在进行优化或裁剪编辑之前：

1. 如果存在 `foundry.toml`，阅读它。
2. 搜索 `src/` 目录下范围内的 Solidity 合约。
3. 默认排除 interfaces，除非用户明确要求包含它们。
4. 在需要理解预期行为时，阅读相邻的测试、mock、fixture、script 和辅助合约。
5. 当 import、继承或库版本会影响优化时，阅读 `remappings.txt`、`lib/`、`node_modules/` 和包配置。
6. 搜索所有可能观察到被修改或裁剪代码的 caller、override、implementation、test、fixture、script、生成客户端和公开 export。
7. 追踪相关数据流：赋值到观察点之间是否经过 return、持久化状态、event / log、外部调用、script 输出、ABI encoding 或测试断言。
8. 追踪相关控制流：guard、modifier、更早的校验、revert 路径、状态机转换、hook、callback 和外部调用。
9. 检查公开契约是否依赖当前形态：ABI 签名、event 字段、custom error selector、revert 行为、storage layout、script 参数、生成产物或部署假设。
10. 检查仓库是否已经存在 benchmark、snapshot、gas-report 或其他优化验证流程。

如果无法检查必要的 caller、合约、依赖、生成产物、storage-layout 参考或验证路径，需要明确说明，不要把优化视为已经完全证明。

### 默认范围

默认检查：

- `src/` 目录下的 Solidity 合约

默认排除：

- interfaces
- vendored dependencies
- 生成代码
- migration 和部署 metadata
- lockfile、snapshot 和纯格式化 churn

如果用户指定了特定合约或文件，则缩小到对应范围。

### 优化类别

检查以下方面的优化机会：

- Gas 效率
- 语义 no-op 裁剪
- 代码结构
- 可维护性

不要机械式优化。只有当改动对每个受支持 caller 和输入都保持预期行为不变，并且收益真实到值得编辑时，才可以修改。

### 语义裁剪类别

主动寻找保持行为不变的删除和简化机会：

- 死赋值和被读取前覆盖的值
- 未使用返回值和未使用输出
- 重复 guard 或已经被强制检查过的条件
- 没有可读性或调试价值的透传临时变量
- private 或 internal helper 参数只是在透传 storage reference，或透传可以从 canonical key 参数重新取得的值
- 在已证明前置条件或状态转换下不可达的分支
- 没有增加边界、不变量、重试、日志、授权、归一化、领域词汇或 Gas 收益的薄 wrapper
- 冗余转换、重复计算或 no-op normalization
- AI 生成代码引入的、没有可达目的的过度防御分支

不要因为代码“看起来没用”就裁剪。只有 caller、数据流、控制流、类型、状态不变量、接口契约和副作用证据证明它不可能影响可观察行为时，才删除。

### 库和版本规则

在替换、简化或微优化与 OpenZeppelin、Solady、Token 标准、代理工具或其他依赖重叠的逻辑前：

- 定位并阅读项目安装的依赖源码。不要凭记忆假设 API、hook 或 storage pattern。
- 相比维护自定义重复逻辑，优先 import、配置或扩展成熟组件。
- 不要把依赖源码粘贴进用户合约作为优化。
- 把继承 hook、必需 override、initializer 顺序、namespaced storage、storage gap 和 state variable 顺序视为优化边界。
- 如果建议的优化可能影响 storage layout、ABI、event 语义、访问控制或外部集成假设，除非用户明确批准更大范围改动，否则保留为建议。

## 优化等级

### 1. 安全的直接编辑

当语义清晰且证明局部完整时，可以直接 patch 低风险优化。

常见示例：

- 在安全的情况下，将 `memory` 改为 `calldata`
- 缓存重复读取的 storage 值或长度
- 删除在任何读取前都会被覆盖的赋值
- 内联只赋值一次、读取一次，且名称不承载领域含义的变量
- 当所有到达 caller 都已经强制相同前置条件时，删除不可达 private 分支
- 当所有 caller 都忽略返回值，且没有 interface 要求时，删除 private 返回值
- 删除冗余计算、变量或分支
- 当 helper 已经接收 canonical mapping key，且每个 caller 都传入同一个 key 的 `mapping[key]` 时，删除 private helper 的 storage-reference 参数
- 删除只调用一个 private helper，且不增加不变量、副作用或有效词汇的 wrapper
- 当行为完全一致时，用已安装且边界清晰的库组件替换本地重复 utility 逻辑

只有在没有公开契约、副作用、诊断行为、storage 预期或框架约定依赖当前代码时，才直接 patch。

### 2. 有条件的编辑

只有在行为清晰且验证强度足够时才 patch。

常见示例：

- 用 custom error 替换 revert string
- `unchecked`
- storage packing
- loop 优化
- state variable 重新排序
- 减少状态转换和外部调用周围的重复读取
- 合并副作用和错误行为等价的重复分支
- 在所有 mutation 路径都已证明后，删除围绕内部状态的防御性检查
- 简化重复 parsing、serialization 或 normalization 逻辑
- 删除面向已文档化不支持版本或模式的兼容分支
- 修改 inheritance、modifier 或 hook 以使用依赖提供的 extension

只有在满足以下条件时才 patch：

- 可以从代码和上下文清楚理解语义
- 不变量来自代码证明，而不是命名推断或当前测试覆盖
- 可观察的 error type、message、log、event、metrics、求值顺序和 revert 行为保持等价
- 不会违反 storage layout 或 upgradeability 假设
- 不会破坏外部集成预期
- 现有验证足够强，可以支撑这个改动

### 3. 只作为建议的编辑

除非用户明确想要更激进的优化，并且该改动有充分理由，否则以下内容只作为建议保留。

常见示例：

- assembly
- 改变 ABI 形态的结构性重写
- 改变公开 ABI、event 形态、custom error selector、CLI/script 输出、序列化字段或生成产物
- 删除 migration、兼容 adapter、feature flag、审计日志、telemetry、rollback 或 cleanup 代码
- 删除不可信输入、授权、资产流动、oracle 假设、callback、重入、并发或锁相关校验
- 裁剪生成代码，或必须匹配 schema、反射系统、部署脚本、decorator、dependency injection container、生命周期 hook 的代码
- 删除用于保护文档化配置范围、版本差异或外部集成的未来兼容代码
- 低价值 Gas 优化，但会明显降低可读性
- 只为了理论收益而进行的大范围多合约重构

对 assembly 要比其他优化技术更加保守。当证明不完整时，保留代码，并说明还缺什么证据。

## 编辑规则

### 先 review，再 patch

在编辑前先识别优化和语义裁剪机会，确保改动集是有意图且范围受控的。

### 先证明，再 patch

对每个非平凡删除或简化，先证明它为什么不会影响可观察行为。证据可以来自 caller 路径、数据流、控制流、类型约束、状态不变量、接口契约、storage 行为、外部调用顺序和副作用分析。

对于 canonical-key helper，需要证明每个 caller 都传入同一个 account、id 或 key 来派生 storage reference，helper 并不是故意操作另一个 mapping entry，并且内部 lookup 会保留 delete、reload 和 aliasing 语义。当 caller 必须在复杂 mutation 中保留预取 slot、操作不同 mapping entry，或避免围绕 `delete` / 重新赋值的 reload 行为时，保留显式 storage reference。

### 只 patch 最小明确收益

只删除或重写已证明优化所需的代码。不要顺手重命名符号、重排文件、重写附近逻辑，或修改触达行之外的格式，除非优化本身需要。

### 保留可观察边界

除非用户明确批准更大范围兼容性改动，把以下内容视为硬性优化边界：

- external/public 函数签名、ABI、export、override、生成客户端和 script/deployment 接口
- storage layout、state variable 顺序、initializer 参数、storage gap、namespaced storage、migration 和 upgradeability 假设
- event / log / error 形态、custom error selector、revert 行为、metrics、tracing 和审计轨迹
- 输入校验、访问控制、auth/session 检查、资产流动、oracle/pair 假设、锁、callback、重入、cleanup 和 rollback
- 框架生命周期 hook、反射、decorator、dependency injection、dynamic import 和生成代码
- 承载业务含义、会计分类、协议状态或团队约定的领域命名

如果代码看起来啰嗦但保护了这些边界，保留它，或者建议用更清晰的注释替代删除。

### 遇到歧义就停止

如果某个优化可能以非平凡方式改变语义、兼容性、诊断信息、执行顺序、持久化、storage 预期、集成假设或可观测性，不要猜测。

应该指出该优化机会，解释风险，并把它保留为建议。

### 保持范围收敛

不要把优化检查变成大型重构、安全审计、功能重写、代码风格清理、formatter pass 或测试重写。用户明确要求时，再把这些工作单独路由。

## Solidity 模式检查

检查 Solidity loop 时，主动寻找低价值的两段式模式：

```solidity
uint256 count;
for (...) {
    if (condition) count++;
}
T[] memory out = new T[](count);
for (...) {
    if (!condition) continue;
    out[index++] = value;
}
```

如果第一轮 loop 只是为了给 event、return value 或本地结果确定 memory array 大小，优先改成单轮 loop：按最大输入长度分配、填入成功项，然后在 emit 或 return 前截断 memory array length。必须保持每个 item 的校验、跳过条件、顺序、重复处理和 event payload 语义完全一致。不要在 count 控制 storage write、授权、定价、外部调用、Gas-critical bound，或 over-allocation 会改变可观察行为的分支里应用这个模式。

## 验证

优先使用项目已有的最强验证路径。

优先级：

1. 现有 benchmark script 或 CI 性能基线
2. 现有 gas snapshot 或 gas 对比流程
3. 现有 gas-report 流程
4. 受影响行为的聚焦测试或回归测试
5. 触达文件的 `forge fmt --check`
6. `forge build`
7. 当触达代码是共享逻辑或外部可见行为时，运行更宽的 `forge test`

如果可用，优先进行优化前后的对比，而不是只做单边测量。只有语义清理验证时，不要编造 Gas 改进。

如果验证失败，需要说明失败看起来是由以下哪类原因导致：

- 优化或裁剪改动本身
- 仓库中已有的问题
- 与 patch 无关的项目配置或依赖问题

如果没有报告验证结果，不要声称优化工作已经完成。

## 输出约定

默认最终输出应包含：

- 已 review 的合约和相关文件 / caller
- 优化和裁剪范围
- 已实现的 Gas changes
- 已实现的语义 no-op 删除或简化
- 每个非平凡删除或简化保持行为不变的证明
- 建议但跳过的优化
- 有意保留的可疑代码
- 风险说明
- 验证结果

优化总结需要分为：

- gas changes
- removed semantic no-ops
- code-structure changes
- maintainability changes
- preserved boundaries
- deferred recommendations

对于跳过的项目，需要包含原因：

- 语义不清晰
- 证明不完整
- 公开契约风险
- 安全或运维边界
- upgradeability 或 storage-layout 风险
- 生成代码或框架拥有的代码
- 兼容性或 migration 风险
- 需要激进的 assembly
- 收益较低，不值得牺牲可读性
- 验证信心不足

## 响应风格

- 直接
- 基于具体文件
- 优先追求真实价值和保持行为不变的删除，而不是聪明压缩
- 不要只用测试作为冗余证明
- 不要编造 Gas 改进
- 不要模糊优化、测试、安全审计和宽泛重构之间的边界
