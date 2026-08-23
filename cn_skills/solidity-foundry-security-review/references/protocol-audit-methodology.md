# 协议审计方法

完整协议审计必须完整阅读本文件。定向审查只读取所选范围可达的章节，并记录哪些攻击面未审查。

## 强制审计顺序

按以下顺序执行，避免实现层漏洞模式干扰协议级风险分析：

1. 协议理解与架构
2. 资产流与托管
3. 角色与信任模型
4. 业务逻辑与状态机
5. Invariant
6. 会计
7. 经济模型
8. Oracle
9. 外部协议集成
10. Access Control
11. 可升级性
12. 签名安全
13. 重入与 Callback
14. 精度与舍入
15. 闪电流动性
16. MEV 与交易排序
17. Token 兼容性
18. 底层 EVM 与 Assembly
19. DoS 与 Griefing
20. Fuzz Testing
21. Stateful Invariant Testing
22. 运维与管理员安全
23. 候选问题验证与 Findings

发现严重问题后不得停止。完成所有适用阶段；受阻或不适用的阶段必须给出具体理由。

## 工作产物

当协议并非平凡合约时，用以下轻量产物作为审计笔记。它们不一定是最终报告标题，但最终回复应暴露足够内容，让结论可复核。

- **入口清单**：外部可达函数、调用者假设、生命周期步骤、移动的价值、修改的关键状态、外部调用和可达特权路径。
- **资产流表**：资产、来源、托管方、会计权威、转换公式/单位、正常去向、应急/特权去向，以及直接转账/Donation 行为。
- **角色矩阵**：角色、直接能力、通过共享 Helper 或 Role Admin 产生的间接能力、最大影响和预期信任假设。
- **Invariant 映射**：Invariant、可能影响它的操作、攻击者可控变量、外部依赖、边界值，以及验证它的测试或推理。
- **外部依赖映射**：依赖项、依赖的服务/价值、谁能影响输出、Revert/Pause/Upgrade/过期行为、Callback 表面，以及失败期间用户能否退出。
- **候选问题台账**：疑似问题、被破坏规则、参与者、前置条件、路径、影响、现有 Guard、证据状态，以及最终归类为 Finding、未解决风险或已排除。

优先使用这些具体轨迹，而不是泛化漏洞 Checklist。只有当某项会影响协议承诺、Invariant、资产流、信任边界或可用性保证时，它才是实质审计点。

## 1. 协议理解与架构

阅读文档、生产合约、接口、库、测试、配置、部署、升级流程和相关已安装依赖。建立紧凑模型：

- 协议目的和预期保证；
- 核心合约及关系；
- 外部可达的生产入口及其所属生命周期阶段；
- 用户角色和特权角色；
- 外部协议和链下参与者；
- 关键资产与状态；
- 主要用户入口；
- 哪个组件持有资产、负责会计、决定价格并可转移价值。

在能够连贯解释主要生命周期和控制边界前，不要开始 Finding 分析。

## 2. 资产流与托管

追踪所有相关原生资产、ERC-20/721/1155、LP Token、Vault Share、Debt Token、Reward Token 和合成资产。

对每种资产确定：

- 来源和入口；
- 每个托管方和转移边界；
- 谁能 Transfer、Mint、Burn、Lock、Approve 或 Rescue；
- 每次转换的公式和单位；
- 所有正常、应急、特权和外部目的地；
- 内部会计是否记录实际资产移动。

追踪 user → vault → strategy → external protocol → token transfer 的完整路径。按适用情况覆盖 deposit、mint、stake、swap、borrow、repay、liquidate、claim、redeem、withdraw、rescue 和 emergency 路径。

寻找资产锁死、超额提现、跨用户错配、未记账余额、重复计价，以及导致资产或会计停留在中间状态的失败路径。

## 3. 角色与信任模型

盘点 owner、admin、operator、manager、guardian、keeper、oracle、upgrader、pauser、minter、signer、multisig、governance、Role Admin 和间接权限。

建立矩阵：

| Role | 直接与间接能力 | 最大影响 | 预期信任假设 |
|---|---|---|---|
| 例：upgrader | 替换实现 | 任意协议行为 | Timelock Multisig |

检查过大权限、缺失验证、权限升级路径、弱权限入口共享内部函数、Role Admin 配置、初始化、权限转移、可抢跑的 Ownership 移交，以及多个有限角色组合后的危险能力。

区分受信任角色的明确设计能力和漏洞。当实现超出信任模型，或破坏该角色本应维持的用户保证时，才构成 Finding。即使属于接受的设计，也要记录集中式信任风险。

## 4. 业务逻辑与状态机

分析完整生命周期，不要孤立审查单个函数。测试正常、重排、重复、跳过、中断和跨用户序列，例如：

- deposit → mint → transfer share → redeem；
- deposit → borrow → price change → liquidate；
- stake → claim → unstake → restake → claim；
- pause → emergency action → unpause → normal exit。

寻找重复领取/提现/记账、状态跳跃、前置条件绕过、Limit/Cooldown/Fee/Pause/Blacklist 绕过、头寸重复使用、重复计价、部分失败，以及合法操作组合后的非法结果。

对每个状态转换记录调用者、前置条件、读取状态、写入状态、移动价值、外部调用和终止条件。

## 5. Invariant

从文档和代码独立推导 Invariant。常见类型：

- 用户不能提取超过其权益的价值；
- 除明确 Fee、Yield、Loss、Mint 或 Burn 外，资产和义务守恒；
- 内部会计与协议选定的权威来源一致；
- 债务、抵押品和偿付能力符合协议规则；
- 累计奖励不超过资金或计划允许值；
- 状态转换不能跳过必要前置条件；
- 特权操作维持文档化用户保证；
- 价格输入新鲜、量纲正确，并能抵抗假定攻击者。

对每个 Invariant，枚举可能影响它的函数、Callback、直接转账、特权操作、极端值和外部状态变化。尽可能把每个实质 Finding 关联到被破坏的 Invariant。

## 6. 会计

识别资产、份额、Supply、Principal、Debt、Borrow、Interest、Reward、Fee 和 Reserve 的权威变量。追踪每次增加、减少、清零、实现和转移。

先确定实际余额还是内部会计是权威来源，再比较两者。重点测试：

- 重复、遗漏或错误清零；
- 外部调用和失败前后的状态更新；
- Transfer 后所有权/会计未同步；
- 直接 Token 转账、强制原生资产和 Donation；
- 时间和用户操作下的利息/奖励累计；
- 重复 Fee，或 Debt 未正确增加/减少；
- Share/Asset 转换，以及首个/最后用户行为；
- 跨用户污染和总量/用户账目对账。

会计差异不自动等于漏洞。确认攻击者能否制造差异、从中获利或用它造成 DoS。

## 7. 精度与舍入

审查所有除法、乘法、百分比、价格/份额转换、Decimals 转换、利息、费用和奖励公式。

确定：

- 是否应先乘后除；
- 每个边界的单位和 Decimals；
- Round Down/Up 对谁有利；
- 拆分或重复操作能否累计利润；
- Dust 能否锁死、被盗或用于攻击；
- 0、1 Wei、最小有效值、边界 ±1、大值和最大支持值的行为。

对 6 位 Token、8 位 Feed、18 位内部精度等混合 Decimals 做明确量纲分析，同时测试单向转换和往返转换。

## 8. 经济模型

分析激励和攻击者盈利能力，不限于代码正确性。考虑有限资本、大额资本、闪电流动性、多地址、高交易次数、有利时机、MEV、任意合约和跨协议组合。

寻找无风险/循环套利、低成本操纵、奖励循环、反向激励、大用户提取小用户价值、杠杆放大、泄漏协议价值的 Tokenomics，以及私人收益超过协议成本的行为。

对候选问题估算攻击本金、攻击成本、可收回本金、协议/用户损失和净利润。区分盈利型提取与攻击者不直接获利但能造成不成比例损害的 Griefing。

## 9. Oracle

将 Chainlink、TWAP、DEX Spot Price、自定义/签名/链下 Oracle、LP Price、Share Price、Fallback 和衍生价格映射到其控制的计算。

检查新鲜度、Heartbeat、Round 完整性、正负/零值、Decimals、Sequencer Downtime、偏差限制、更新延迟、Pause/Fallback、权限、来源切换、循环依赖和单位一致性。

除非协议证明安全，否则把 DEX Spot Price 视为可操纵。测试 Flash Swap、大额交易、低流动性、Donation、LP 操纵和同交易读写影响。追踪到抵押品、借款额度、清算、Mint/Redeem、份额和奖励。

## 10. 外部协议集成

把每个外部协议和 Token 调用视为信任边界。阅读实际安装的接口和实现假设。

检查返回值、Revert、Callback、Slippage、Deadline、Allowance 生命周期、价格、Decimals、Pause/Upgrade、可变外部状态、部分完成和异常返回数据。

对每个依赖回答：

- 协议依赖它提供什么服务或价值；
- 哪些输出影响资金或会计；
- 谁能影响这些输出；
- Revert、Pause、状态过期、行为变化或升级时会怎样；
- Callback 能否观察或重入中间状态；
- 依赖失败时用户资金能否退出。

## 11. Access Control

盘点所有改变关键状态的外部可达函数，尤其是 Setter、Update、Upgrade、Withdraw、Rescue、Mint/Burn、Pause/Unpause 和 Initialize。

检查 `onlyOwner`、`onlyRole`、Role Admin、自定义 Modifier、`msg.sender`、`tx.origin`、代理上下文、Callback、Meta-Transaction、不同权限入口共享的内部 Helper，以及绕过预期边界的间接调用。

验证谁能 Grant、Revoke、Renounce、Transfer 或恢复权限，以及两步/延迟转换能否绕过或永久阻塞。

## 12. 可升级性

对 Transparent、UUPS、Beacon、Diamond 和自定义 Delegatecall Proxy，检查：

- Initializer/Reinitializer 可达性和版本；
- Implementation 初始化和 Constructor 假设；
- 升级授权、Proxy Admin 和目标验证；
- Delegatecall 上下文和任意执行路径；
- Storage Layout、Collision、Gap、Namespaced Slot、继承顺序和类型变更；
- 迁移逻辑和跨版本 Invariant；
- 升级是否改变会计含义或权限边界。

解析实际代理模式和依赖源码。不要假设 Constructor 部署与 Proxy 初始化等价。

## 13. 签名安全

对 EIP-712、Permit、Authorization、Meta-Transaction 和链下订单，验证签名摘要绑定所有安全关键参数：Signer、Action、Amount/Token/Position、Receiver、必要时的 Caller、Nonce、Deadline、Chain ID、Verifying Contract 和 Domain/Version。

检查 Nonce 唯一性与消耗时机、取消、过期、签名可塑性、合约签名者、跨链/跨合约/跨操作 Replay、Domain Separator 变化和部分成交语义。

## 14. 重入与 Callback

分析单函数、跨函数、跨合约和 Read-only Reentrancy。追踪状态或会计处于中间态时的每个外部调用，包括 Token Hook、ERC-721/1155 Receiver、DEX/Flash Callback、Fallback/Receive 和任意用户目标。

单个函数有 `nonReentrant` 不代表安全。确认 Callback 能否在 Invariant 恢复前进入其他函数、其他协议组件、被外部协议读取的 View 或特权 Callback 表面。

## 15. 闪电流动性

把闪电流动性视为攻击者能原子使用巨额资本，而不是漏洞本身。用临时大余额重新检查 Oracle、Voting、Reward、Share Price、Liquidity、Collateral、Liquidation、Accounting 和 Token Price。

测试 borrow → manipulate → trigger protocol action → unwind → repay。同交易现实攻击不能仅因为使用闪电流动性而被忽略或降级。

## 16. MEV 与交易排序

分析 Swap、价格更新、清算、奖励领取、拍卖、Deposit、Withdraw 和 Mint 周围的 Front-run、Back-run、Sandwich 和任意排序。

检查 Slippage、Deadline、过期 Quote、用户控制的最小输出、适用时的 Commit/Reveal，以及攻击者看到 Pending Transaction 后能否在前后执行获利。区分不可避免的市场 MEV 与协议制造的价值损失或保证破坏。

## 17. Token 兼容性

不要假设 `amount sent == amount received` 或所有 ERC-20 行为一致。按适用情况考虑 Fee-on-transfer、Rebasing、ERC-777/Hook、无返回值/返回 False、Blacklist、Pause、Upgrade、Callback、异常 Decimals、转账限制，以及对零值或特定 Approve 方式 Revert 的 Token。

检查 Safe Transfer Wrapper、Allowance 生命周期、Balance Before/After、Rebase 下的 Share Accounting、Blacklist/Pause 导致的退出 DoS，以及无权限 Token Listing 是否暴露不支持行为。只要求协议承诺支持的兼容性。

## 18. 底层 EVM 与 Assembly

提高对 Assembly、`call`、`delegatecall`、`staticcall`、CREATE/CREATE2、原始 Storage、手工 ABI 编码和 Returndata 解析的审查强度。

检查 Calldata Offset、Free Memory Pointer、Memory 覆盖、Returndata Length/Copy、Dirty Bits、Selector、Value 转发、Revert 传播、Storage Slot、Address Mask 和编译器假设。

对 Delegatecall，确认执行谁的代码、使用谁的 Storage、有效 Sender/Value、目标是否可控，以及 Storage Layout 是否匹配。证明语义等价，不要默认 Assembly 只是高级 Solidity 的 Gas 优化版本。

## 19. DoS 与 Griefing

检查无界循环、攻击者控制的数组/Storage 增长、恶意 Revert、Gas Exhaustion、Dust Position、强制转入、Blocked Recipient、Callback、全局锁和外部依赖失败。

衡量攻击成本、可重复性、影响范围、持续时间和恢复路径。即使攻击者不直接获利，低成本阻塞整个协议也可能是实质性问题。区分单用户自我 DoS 与协议级可用性损失。

## 20. Fuzz Testing

对 deposit/withdraw、mint/redeem、borrow/repay/liquidate、swap、claim、stake、unstake 等关键数学和状态转换使用 Foundry Fuzz Test。

围绕安全性质设计 Assertion，不要只追求随机输入数量。覆盖 0、1、最小有效值、边界 ±1、大值、最大值、Decimals 组合和有意义的关联输入。Fuzz 发现异常后，缩减为确定性回归 Case，并在称为漏洞前分析 Root Cause 和 Impact。

## 21. Stateful Invariant Testing

用 Handler 模拟真实的多用户、多函数、长序列行为。按适用情况包括 deposit、withdraw、mint、redeem、borrow、repay、transfer、claim、stake、unstake、liquidation、时间变化、价格变化和特权操作。

优先验证资产守恒、用户权益、Share/Debt/Reward/Fee Accounting、Solvency、协议余额和可恢复性。必要时用 Ghost Variable 表达预期净流量。Bound 输入时不要排除正在审查的边界条件。

## 22. 运维与管理员安全

评估 Pause、Emergency Withdraw、Oracle Fallback、Rate/Withdraw Limit、Multisig、Timelock、Monitoring 假设、Upgrade Delay 和恢复流程。

模拟 Admin Key 泄露、Oracle 过期、外部协议 Pause/Upgrade、DEX 无流动性、Token Blacklist/Pause 和 Multisig 失陷。验证 Pause 后哪些操作仍可用、用户能否退出、Emergency 路径是否保留会计、Timelock 能否绕过，以及 Guardian/Rate Limit 是否真正限制损失。

## 23. 候选问题验证与 Findings

没有 Root Cause、可达攻击/失败路径和 Impact，不得把候选问题正式化。验证依赖的准确行为和所有现有 Guard。对实质性问题优先提供最小 Foundry PoC、具体状态轨迹、数值示例或数学证明。

Severity 由 Impact 和 Likelihood 决定，并结合攻击成本、权限、资本、受影响资产/用户、利用复杂度和可恢复性。不要把代码风格报告为安全问题，不要把测试失败直接等同于漏洞，也不要把未证明理论标为 High/Critical。

## 完成 Checklist

完整协议审计只有在每项完成，或基于证据明确标记为受阻/不适用后才能结束：

- [ ] 已解释协议目的、架构、组件和主要生命周期。
- [ ] 已盘点生产合约、入口、依赖、部署/配置和升级路径。
- [ ] 已追踪关键资产、托管位置、转移路径和退出/应急路径。
- [ ] 已记录角色、间接权限、信任假设和最大影响。
- [ ] 已审查业务流程、状态机、跨函数组合和失败路径。
- [ ] 已独立推导关键 Invariant，并映射到状态修改操作。
- [ ] 已对账 Asset/Share/Debt/Reward/Fee/Reserve Accounting。
- [ ] 已检查精度、Decimals、舍入方向、Dust 和边界行为。
- [ ] 已考虑经济激励、攻击成本/利润、大资本和跨协议组合。
- [ ] 已审查 Oracle 来源、操纵、新鲜度、单位、Fallback 和下游影响。
- [ ] 已审查外部集成、Callback、失败行为、Pause 和升级假设。
- [ ] 已审查访问控制、角色转换、初始化和间接绕过。
- [ ] 适用时已审查升级授权、Implementation 初始化、迁移和 Storage Layout。
- [ ] 适用时已审查签名绑定、Nonce/Deadline/Domain 和 Replay 保护。
- [ ] 已考虑单函数、跨函数、跨合约和 Read-only Reentrancy。
- [ ] 已考虑闪电流动性和交易排序/MEV 影响。
- [ ] 已测试协议支持的 Token 行为和兼容性假设。
- [ ] 存在时已审查底层 Call、Assembly、Delegatecall、Storage、Memory 和 Returndata。
- [ ] 已分析 DoS/Griefing 成本、范围、持续时间和恢复。
- [ ] 已执行或具体设计关键数学与边界的 Fuzz Test。
- [ ] 已对复杂协议执行或具体设计 Stateful Invariant。
- [ ] 已审查 Pause、Emergency、Timelock、Multisig、Monitoring 和恢复假设。
- [ ] 每个正式 Finding 都有受影响代码、Root Cause、路径、Impact、修复和回归测试。
- [ ] 每个 High/Critical Finding 都有可验证路径或 PoC。
- [ ] 已如实记录实际执行的测试和结果。
- [ ] 已记录排除项、不可用证据、未解决经济风险和未验证的信任/设计假设。

## 禁止的捷径

不得：

- 因为存在 `nonReentrant`、知名库或通过的测试就推断安全；
- 只审查单个函数，忽略可达的跨函数/跨合约行为；
- 忽略资金、会计、经济、舍入、管理员能力或外部依赖；
- 假定 Token 标准、Oracle 正确、Callback 诚实或外部集成永久可用；
- 假定管理员诚实，除非这是明确的 Trust Assumption；
- 把无法证明的问题报告为漏洞，或在没有路径与影响时抬高 Severity；
- 仅因为需要闪电流动性或复杂交易序列就降低 Severity；
- 为了满足测试而改变协议预期行为；
- 声称并未发生的审查覆盖或命令执行。
