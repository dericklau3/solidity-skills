---
name: foundry-test
description: "在 Solidity / Foundry 开发完成后审查并强化测试。用于添加或改进高价值 Unit、Fuzz、Invariant、集成或真实流程测试，维护 `test/docs/*.md`，并报告 `forge test` 验证结果。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# 审查并强化 Foundry 测试

## 核心工作流

### 回答前先理解请求

对于概念问题（例如“这个合约应该怎么测？”），只解释思路，不修改代码。对于添加、审查、改进或记录测试的请求，执行下面的流程。

### CRITICAL: 总是先阅读项目

在编写或修改测试之前：

1. **搜索用户项目**中的 Solidity 合约、Foundry 测试、部署脚本、辅助工具、mock、fixture 和 `test/docs`。
2. **阅读相关文件**，理解现有行为、测试风格、setup 模型和文档约定。
3. **默认融入现有测试套件，而不是替换它**。按本地风格添加聚焦的测试和 helper。只有用户明确要求时，才大范围重组或替换测试结构。

如果某个文件无法读取，明确说明失败。报告尝试读取的路径和原因。不要在缺少项目上下文时假装可以给出完整测试结论。

### 基本规则：测试行为，而不是测试数量

在添加任何测试前，先确认它证明的行为或保证：

1. **存在明确行为缺口？** 添加或改进能证明它的最小测试。
2. **行为已经被覆盖？** 不要为了增加测试数量而添加重复测试。
3. **预期行为不明确？** 停下来指出歧义，不要编造断言。

优先编写少量高信号测试，而不是机械扩充覆盖率。

### 依赖规则：测试项目的真实集成方式

当合约继承或组合 OpenZeppelin、Solady、Chainlink、Uniswap、代理库或其他依赖时：

1. 通过 `foundry.toml`、`remappings.txt`、`lib/`、`node_modules/` 或包配置定位已安装源码。
2. 当行为依赖 hook、modifier、role check、initializer、token callback、oracle 语义或可升级性时，阅读对应依赖组件、mock、example 或 test。
3. 可以把依赖示例和生成的 baseline 作为集成形态参考，但项目测试应聚焦用户自己的行为和 invariant。
4. 不要整段复制依赖测试，除非项目明确 vendor 这些测试。
5. 当风险涉及必需 override、initializer 顺序、storage 兼容性或相互作用的 extension 时，添加对应回归测试。

### 方法论

主要流程是**从项目源码和现有测试中发现行为**：

1. 检查合约、测试、脚本、文档、mock 和 helper。
2. 当 import 行为影响预期结果时，检查依赖源码。
3. 识别 entrypoint、资产、角色、accounting 关系、权限、状态转换，以及理论上不应出现的状态。
4. 为每个行为选择能证明它的最轻测试层级。
5. 同步修改测试和对应的 `test/docs`。
6. 运行 `forge test` 并报告结果。

完整流程见 [行为发现和测试强化](#行为发现和测试强化)。

## 行为发现和测试强化

这个流程用于强化 Foundry 测试，但不把任务变成重写、审计或覆盖率最大化工程。

**前置条件：**始终遵守上面的行为优先规则。

### Step 1: 识别测试表面

1. 如果存在，阅读 `README.md` 和 `foundry.toml`。
2. 在 `src/` 中搜索范围内的合约。
3. 阅读 `test/` 下的相关测试。
4. 当行为依赖部署配置时，阅读 `script/` 下的部署或 setup 脚本。
5. 如果存在，阅读 `test/docs/*.md`。
6. 当依赖行为重要时，阅读 `remappings.txt`、`lib/`、`node_modules/` 和包配置。
7. 只在需要理解当前 setup 时，阅读 mock、fixture 和辅助合约。

### Step 2: 将行为映射到测试层级

使用能证明该行为的最轻层级：

1. **单元测试** - 适合在指定条件下验证一个具体操作：返回值、状态更新、资产流转、事件、边界检查、失败路径和访问检查。
2. **集成测试** - 适合依赖多个合约、部署设置、外部交互、资产流转或真实顺序的行为。
3. **真实流程测试** - 适合风险来自用户或操作方如何通过 public entrypoint 到达该行为的场景。
4. **Fuzz 测试** - 适合大输入范围下的算术、校验、单调性、幂等性或状态转换。
5. **Invariant 测试** - 只用于跨调用序列长期成立的系统属性。

记住层级边界：Unit Test 测具体操作，Fuzz Test 测参数空间，Invariant Test 测协议状态空间。

不要把每个单元行为都升级成集成测试。不要机械生成 fuzz 或 invariant 测试。

对于依赖支撑的行为，添加能证明项目集成点的最窄测试：override 被调用、role modifier 正确限制路径、initializer 不能跳过或重复、callback 被安全处理，或库施加的 invariant 在用户流程后仍然成立。

测试优先级应从资产和状态风险出发：

1. 资金安全和资产流转。
2. Accounting 和偿付能力。
3. 权限和特权操作。
4. 核心状态转换。
5. 数学、取整、share、价格、比例和 BPS。
6. 边界条件。
7. 普通业务逻辑。

优先覆盖可能造成资金损失、无限 mint、资金锁死、无权限提款、accounting 漂移、协议资不抵债、重复领取、费用绕过或价格错误的场景。

### Step 3: 设计 Unit Test

对每个重要的 `public` 或 `external` 状态修改函数，判断是否需要单元测试。常见对象包括 `deposit`、`withdraw`、`swap`、`stake`、`unstake`、`claim`、`borrow`、`repay`、`liquidate`、`mint` 和 `burn`。

高信号 Unit Test 通常覆盖：

- Happy path：预期操作由正确调用方成功执行。
- State change：核心 storage、supply、share、position、reward 和全局总量变化正确。
- Asset movement：用户、协议、treasury、fee receiver 和其他接收方余额变化正确。
- Revert conditions：零金额、余额或 allowance 不足、权限不足、超过最大值、非法状态、重复操作、非法地址、deadline 过期或协议特定非法输入。
- Access control：授权调用方成功，未授权调用方使用预期 custom error 或 selector revert。
- Boundary：`0`、`1`、最小允许值、最大允许值、最大值附近、余额或 allowance 刚刚足够，以及差 1 不足。
- Events：重要状态修改操作发出预期 event，并检查有意义的 indexed 和非 indexed 值。

如果可以检查状态、余额、事件或 accounting，不要写只证明“不 revert”的测试。

### Step 4: 设计 Fuzz Test

当风险隐藏在大量输入组合中，而不是某几个手写样例里，使用 Fuzz Test。重点对象包括：

- fee、reward、share、exchange rate、price、interest、percentage 和 BPS 计算。
- deposit、withdraw、swap、stake、borrow、repay 和 claim 数量。
- lock duration、vesting time、reward duration、deadline、epoch 和时间相关状态。
- slippage、collateral ratio、discount、reward weight 等比例。

优先使用 `bound()` 将输入限制在有意义的协议范围内。谨慎使用 `vm.assume()`；如果大部分随机输入都被过滤掉，应重新设计输入模型，使用 bound、actor 选择或 state-aware setup。

Fuzz Test 应验证 property，而不是把单元测试换成随机数字。常见 property：

- fee 始终满足 `0 <= fee <= amount`。
- deposit 按协议取整规则增加协议资产和用户 share。
- 在没有费用、收益或超出协议设计取整的情况下，deposit 后 withdraw 能取回应得资产。
- 输入增加时，结果按预期单调变化或满足对应关系。
- 非法参数范围使用预期 error revert。

### Step 5: 设计 Invariant Test

只有在理解协议并覆盖重要单操作行为后，才添加 Invariant Test。不要从 invariant 开始，也不要不经当前业务推导就复制其他协议的 invariant。

从协议真实承诺中推导 invariant：

- **Accounting：** 内部记账与真实资产或用户余额总和一致。
- **Solvency：** 协议控制资产足以覆盖用户 claim。
- **Conservation：** 除非 mint、burn、fee、reward 或损失规则明确允许，资产不能凭空出现或消失。
- **Supply：** minted 减 burned，或用户 share 总和，与报告的 total supply 一致。
- **Balance：** 用户和协议余额不能进入不可能或制造债务的状态。
- **Permission：** 任意调用序列后，非特权 actor 不能获得管理员能力或抽走特权资金。
- **State machine：** paused、closed、liquidated、claimed、locked 或 expired 等状态不能被绕过。
- **Economic：** 反复小额操作、donation、first depositor、rounding、share inflation 或 fee bypass 不能按协议规则之外抽取价值。

做 Stateful Invariant Testing 时，创建只暴露真实用户或操作方关键行为的 Handler。使用 `targetContract(address(handler))`，需要收窄随机调用集合时使用 `targetSelector()`。

Handler 设计规则：

- 生成有效但多样的操作，避免几乎所有调用都 revert。
- 用 `bound()` 和 state-aware action guard 让调用有意义，但不要隐藏 bug。
- 协议有多用户交互时，模拟多个 actor，并通过输入 seed 或本地 helper 选择 actor。
- 当协议自身状态不足以检查 accounting 时，使用 ghost variables 记录 deposited、withdrawn、fees、rewards、minted 或 burned。
- Invariant 函数应聚焦核心协议属性；少量有意义的 invariant 优于大量平庸断言。

Invariant 失败时，根据 Foundry 输出的调用序列缩小到最小真实复现路径。在协议行为或测试假设被解决前，保留失败测试或复现用例。

### Step 6: 优先使用项目真实 setup

当项目有可复用 setup 或部署 helper 时，优先使用它们，让测试覆盖项目预期的输入、顺序、环境假设和部署后配置。

只有当 shortcut 本身不是被测行为时，才使用 mock 或直接 helper。如果测试声称证明真实用户路径，动作应通过相关 public entrypoint 完成。

对于 fork 测试，尽可能固定 block，并记录会影响断言的假设。

### Step 7: 修改测试

只有在缺失行为明确后，才添加或修改测试。

保持修改聚焦：

- 复用本地命名、fixture、helper 模式和断言风格。
- 只有当新 helper 能减少有意义的重复时才添加。
- 除非测试必须且属于用户请求范围，否则避免生产代码重构。
- 只有当外部数学、取整、时间或单位转换导致精确相等不合适时，才使用近似断言。
- 根据需要使用 Foundry 工具，例如 `vm.prank`、`vm.startPrank`、`deal`、`vm.deal`、`vm.warp`、`vm.roll`、`vm.expectRevert`、`vm.expectEmit`、`bound`、`vm.assume`、`targetContract` 和 `targetSelector`，但优先复用项目已有 helper。

如果新增测试失败，不要立刻削弱断言或改测试让它变绿。先判断是测试假设错误，还是协议行为错误。如果疑似协议错误，保留可复现测试，记录触发条件、预期行为、实际行为、受影响函数或资产，以及风险。

不要通过扩大 tolerance、添加 `assume()` 排除失败输入，或擅自修改生产行为来隐藏真实问题；除非用户确认协议 bug 且修复属于当前范围。

### Step 8: 维护测试文档

当添加或实质修改测试文件时，在 `test/docs` 下创建或更新对应文档。

默认约定：

- `test/Foo.t.sol` -> `test/docs/FooTest.md`

如果仓库已有其他约定，遵循本地约定。

每份测试文档应记录：

- 测试文件的目的和 setup 模型
- 主要场景或测试组
- 重要 helper 以及它们准备了什么
- 重要的 fork、mock、时间、取整或环境假设

文档应描述行为，不要逐行复述断言。

## 验证

编辑后，运行 `forge test`。

如果聚焦验证有帮助，先运行窄范围命令；可行时再运行完整测试套件：

- `forge test --match-path <path> -vvvv`
- `forge test --match-test <testName> -vvvv`
- `forge test`

如果 invariant 失败，检查 Foundry 输出的失败调用序列，并在可行时缩小到最小复现。

如果缺少 RPC 访问、依赖项或环境变量，报告具体阻塞原因和无法完成的命令。

不要在没有报告验证结果的情况下声称完成。

## 输出约定

默认最终输出应包括：

- 已审查文件
- 添加或更新的测试
- 添加或更新的测试文档
- 添加或改进的 Unit Test，包括覆盖的操作和保证
- 添加或改进的 Fuzz Test，包括参数范围和验证的 property
- 添加或改进的 Invariant，包括为什么必须成立、如何验证、涉及哪些合约
- 有意跳过的缺口及原因
- `forge test` 结果
- 测试过程中发现的潜在协议问题，包括位置、触发条件、预期结果、实际结果、风险和复现测试
- 剩余歧义或风险
