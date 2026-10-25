---
name: develop-secure-contracts
description: "用于通过 OpenZeppelin 权限、重入保护、紧急暂停、ERC20、ERC721 或 UUPS 升级组件开发 Solidity 合约，严格执行入参校验、资金流顺序与合法调用流程。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# 使用 OpenZeppelin 开发合约

## 执行约定

使用适用的 OpenZeppelin 组件实现用户要求的合约。遵守本技能的校验、资金流、状态转换和验证规则，完成已授权实现，不停留在建议。用户明确指令及更高优先级指令优先。受阻步骤如实报告，继续独立工作。

编辑前读取项目配置、目标代码、测试、部署脚本和实际安装依赖，保留无关改动。使用实际编译器/backend 确认支持 `require(condition, CustomError(...))`，不设固定编译器版本、不静默升级，也不擅自改成 revert string 或 `if/revert`。存在不兼容时，先解决再实现依赖该语法的部分。

增量开发已有系统时，保留请求范围外的函数/event ABI、错误约定、权限、资产语义和存储兼容性。不为统一风格改写已有入口。完成请求确实需要改变这些契约时，先识别受影响的 caller、集成和部署状态，仅在已有授权范围内实施并同步相关依赖；需要扩大范围的部分说明具体原因，继续独立工作。

## 沿用 foundry-starter 编码风格

默认按用户的 foundry-starter 风格开发。编写或修改 `src/`、`script/` 前，读取[编码与部署约定](references/foundry-starter-style.md)，并对照当前项目的 Wallet、Errors 和部署脚本等价文件。先确认导入、命名、错误组织、函数结构及部署职责，再实现业务；不能只满足安全规则却重新设计项目组织。

源码优先复用共享 `Errors` 和现有命名习惯；部署沿用外层入口控制广播、`ContractCollection` 接收明确参数并完成部署、`BaseDeployScript` 管理配置、`ContractRegistry` 保存地址的分工。项目明确约定和用户指令优先；模板中的业务限制、地址、依赖版本与逻辑缺陷不得机械复制。

## 按业务流程组织函数

函数按业务职责和调用关系就近组织，让读者从入口向下连续读到实现细节。新增或修改函数时采用“业务入口 → 该入口的 internal/private helper → 下一组业务函数”的顺序；多层专用 helper 按调用链向下排列。函数可见性不作为全文件分组依据，不把所有 external/public 入口放在前面、所有 internal/private helper 统一堆在末尾。

- 仅供一个入口使用的 helper 紧跟该入口，避免在两者之间插入无关配置、查询或其他业务函数。配置入口及其 `_setXxx` / `_updateXxx` 同样成组。
- 多个入口共用的 helper 只保留一份，放在主要业务入口之后或同一业务组的末尾；初始化也调用的配置 helper 仍放在对应配置入口附近。跨业务组复用时，以主要职责确定归属，不为满足相邻要求复制逻辑。
- 例如 FomoPool 的 `depositRewards` 调用 `_rolloverRound`，应将 `_rolloverRound` 放在 `depositRewards` 后面，再放 `updateRewardRatios` 与 `_setRewardRatios` 等配置组；`publishResults` 继续复用这一份 `_rolloverRound`。不要让无关配置函数隔开入金入口与轮次结转实现。
- 这项约定用于新增代码及本次修改涉及的函数组，不自动扩大为整份合约重排。调整位置时连同 NatSpec 和解释注释一起移动，保持函数逻辑、可见性、modifier、ABI、继承及状态变量布局不变。

## 命名、权限与配置写入约定

以下是新增合约和本次授权修改代码的默认约定；参考项目中的旧写法不覆盖这些规则。已有 ABI、权限或存储兼容性仍按前述增量开发边界处理，不为统一风格改写范围外代码。

- `constructor` 和 `initialize` 的所有具名参数统一使用尾下划线，例如 `token_`、`receiver_`、`amount_`；不以是否与状态变量同名为条件。`owner_` 仅说明命名形式，构造和初始化入口不接收 owner 参数。
- 权限使用 `Ownable`，代理合约使用对应的 `OwnableUpgradeable`；不主动引入 `Ownable2Step`、`Ownable2StepUpgradeable` 或 `AccessControl`。普通合约按实际依赖版本通过 `Ownable(msg.sender)` 或该版本等价机制设置 owner；代理在 `initialize` 中通过 `__Ownable_init(msg.sender)` 或该版本等价机制设置 owner。
- owner 取执行构造或初始化时的 `msg.sender`，不使用 `tx.origin`，不额外传入 owner/initialOwner/admin 来指定 owner。部署和测试核对真实调用者；通过工厂创建时 owner 可能是工厂，不能假定总是最外层 EOA，也不能为调整 owner 偷加初始化参数。
- 所有 `constant`、`immutable` 状态变量都使用 `private`，例如 `uint256 private constant _MAX_AMOUNT = 100e18;`、`IERC20 private immutable _token;`。仅在业务明确需要外部读取时提供显式 getter，不通过 `public` 声明自动生成 getter。这项限制不适用于普通状态变量；代币、Router、主池等需要外部核对的普通配置使用 `public`，不为封装而一律改成 private 或额外编写同义 getter。`private` 不会隐藏链上数据，不把它当作保密措施。
- 所有 `private` 状态变量名称都以 `_` 开头：普通变量和 immutable 使用 `_camelCase`，constant 使用 `_UPPER_SNAKE_CASE`，例如 `_token`、`_mainPair`、`_MAX_AMOUNT`。沿用已有 `__gap` 等升级保留字段，不为命名统一改动已有升级存储标识；用户明确要求的命名修正除外。
- 在 `constructor` 或 `initialize` 中设置、且另有外部配置入口可修改的状态，必须抽成共享 `internal` 函数 `_setXxx` 或 `_updateXxx`。初始化和外部入口调用同一 helper，由 helper 统一负责入参校验、状态写入和配置事件；外部入口保留 `onlyOwner` 及适用 guard。不可在初始化中直接赋值、在 setter 中再重复一份逻辑，也不通过 `this.setXxx(...)` 绕行外部调用。仅初始化时写入且没有外部修改入口的变量可以直接赋值。
- 地址参数的有效性 `require` 只检查 `!= address(0)`，不添加 `xxx.code.length > 0`、`extcodesize` 或 `Address.isContract` 等合约代码存在性限制。业务明确用零地址表示特殊模式时保留该语义。权限、支持资产等业务条件仍单独校验；第三方协议身份与部署核验在配置和 Fork 验证中完成，不转化为入口代码长度检查。

共享配置写入示例（片段，沿用项目的 `Errors` 和事件定义）：

```solidity
function initialize(address receiver_) external initializer {
    __Ownable_init(msg.sender);
    _setReceiver(receiver_);
}

function setReceiver(address receiver_) external onlyOwner {
    _setReceiver(receiver_);
}

function _setReceiver(address receiver_) internal {
    require(receiver_ != address(0), Errors.InvalidReceiver(receiver_));
    address previous = receiver;
    receiver = receiver_;
    emit ReceiverUpdated(previous, receiver_);
}
```

## 第三方协议使用目标网络真实部署

项目集成第三方链上协议时，必须调用目标网络的真实协议部署，包括但不限于 Uniswap、PancakeSwap、Aave。该要求适用于所有实际集成的第三方协议，不限于列举名称；不要自行实现简化版或部署本地仿制协议来替代。

- 实现前确认目标 chain ID、协议版本、官方部署地址及项目使用的具体市场/池。核对官方部署资料、项目配置和链上代码/关键配置，不能仅凭地址非零或存在代码就认定是正确协议，也不能混用不同链或版本的组件。
- 使用匹配实际部署版本的接口或官方依赖编写调用。可编写项目所需的 adapter、策略和回调合约，但其底层必须调用真实协议；不得在 adapter 内用自建逻辑伪造协议结果。
- 协议地址与市场参数沿用 `BaseDeployScript` 的链配置组织方式。部署脚本连接现有协议，不为方便部署替身 Router、Factory、Pool 或借贷市场。业务需要新池/头寸时，通过目标链真实协议提供的入口创建。
- 集成验证必须在目标网络固定区块 Fork 上执行真实调用，遵循 `foundry-test` 的真实协议要求。不得用 mock 通过作为交付依据；本地 Fork 验证不代表已广播真实交易。
- 目标链没有所需协议版本，或地址、RPC、历史状态和配置无法核实时，明确报告缺口，继续独立工作；不静默换链、换版本或改成本地仿制协议。

该规则针对第三方链上服务的集成；OpenZeppelin 等源码组件仍按实际安装版本导入或继承，不要求把源码库改成外部协议调用。

## 使用 OpenZeppelin 组件

按请求的行为选用组件，不为每个合约机械添加全部组件。导入实际安装源码，不复制源码或重复实现已有基础能力。根据该版本确认路径、API、hook、constructor 和 initializer。

| 需求 | 集成规则 |
| --- | --- |
| 权限 | 使用 `Ownable`，代理使用 `OwnableUpgradeable`；owner 由构造或初始化中的 `msg.sender` 确定，不接收 owner 参数。按需限制 mint、配置、暂停/恢复、资产回收和升级。不无故叠加权限体系。 |
| 重入保护 | 对资产转移入口，以及共享暴露会计状态的其他可写入口，直接使用 OpenZeppelin `ReentrancyGuardTransient` 和 `nonReentrant`；普通合约和代理合约均遵循此规则，不默认改用 `ReentrancyGuard` 或其 Upgradeable 版本。确认已安装依赖提供该组件、编译配置及目标网络支持 EIP-1153 transient storage；不满足时明确报告兼容性阻塞，不静默替换组件或升级依赖。受保护的 external 函数复用 internal helper，避免相互调用导致嵌套进入重入锁。 |
| 紧急暂停 | 使用 `Pausable` 与有权限的暂停/恢复入口。明确暂停哪些操作、是否保留紧急退出。仅继承不会自动暂停：需使用 guard 或适当的 token pause 扩展/hook。 |
| ERC20 | 继承 `ERC20` 及必要扩展，外部代币转账使用 `SafeERC20`。保留标准 transfer、allowance、mint/burn 和 event 语义。 |
| ERC721 | 继承 `ERC721` 及必要扩展。将 safe transfer 和 safe mint 视为回调边界，验证接收处理与所有权规则。 |
| UUPS | 使用已安装版本的 UUPS 实现模式与 `ERC1967Proxy`，需要时选用支持升级的有状态组件。通过 `_authorizeUpgrade` 限定升级权限。 |

UUPS 部署代理时原子完成初始化，按正确顺序对必需父级各初始化一次，并禁用实现合约的 initializer。Reinitializer 仅用于明确的迁移。保持已有代理 storage、继承、namespaced storage 和升级授权。供已有代理使用的新实现，即使尚未部署也属于升级。宣称布局兼容前，对照已部署基线或运行项目升级验证器。

## 函数入口校验

每个实现函数一进入就使用 `require(validCondition, CustomError(...))` 校验适用的入参前置条件，位于业务状态修改或资产转移调用之前。可在入口调用校验 helper 或使用包含这些校验的 modifier。Internal helper 在入口建立前置条件；所有可达 caller 已证明执行同一校验且使用前条件不会变化时，不重复检查。

按业务规则验证地址角色、支持资产、金额、数组长度/边界、ID、期限、限额和参数间关系。不编造非法输入：例如 ERC20 合法的零额转账、合法的 token ID 零仍需允许。保留 OpenZeppelin 自带权限/guard 校验，不为统一语法重写库内部。

```solidity
require(receiver != address(0), InvalidReceiver(receiver));
require(amount > 0, InvalidAmount(amount)); // 仅适用于要求正数金额的操作。
require(amount <= available, InsufficientBalance(available, amount));
```

按项目错误组织方式声明或复用具体 custom error。错误参数必须低成本、无副作用：`require` 成功时也会求值。索引前先检查边界，执行可能先失败的算术前先检查条件。依赖状态的校验使用权威状态。只有转账后才能获知的条件，还需在转账后校验。

## 资金流顺序

区分业务会计状态、重入锁和必要的最小回调认证上下文。外部交互前必须进入重入保护；写入保护状态不属于提前给存款记账。

### 资金流入：校验 → 收到资产 → 验证到账 → 修改业务状态

- 先校验入参、权限、暂停状态及合法状态转换。
- 转入资产前启用重入保护。ERC20 使用 `safeTransferFrom`；ERC721 使用预定的安全托管转账。到账成功前不得增加余额、份额、存款或奖励。
- 明确采用足额到账还是支持 fee-on-transfer。足额托管需确认余额增量等于请求数量，不一致立即回退。明确支持转账税时，按实际到账量记账并检查适用的最小到账要求。不得按未经验证的请求数量记账。
- ERC721 托管需认证预期 token 合约、operator/from/token ID 和回调上下文，转账后确认所有权。必需的接收回调仅确认预期转账，不提前计入存款、不开放其他业务入口；非预期或重复回调立即回退。若给必需回调添加 `nonReentrant` 会阻止受保护的存款完成，则改用回调认证并禁止业务状态修改。
- 原生资产 payable 入口在函数体执行前已由 EVM 收到 `msg.value`；先验证 `msg.value` 再更新会计，无需再次转入。直接捐赠或强制转入的余额不得自动计入用户资产。
- 验证到账后再更新所有相关会计并发出成功事件。不得让回调或依赖 view 的外部组件使用资产与会计不一致的快照；保护敏感读取，或避免向集成方暴露该中间状态。

### 资金流出：校验 → 修改业务状态 → 转出资产

- 校验接收者、权益、金额、流动性、权限、暂停规则及状态转换。
- 在重入保护下，先扣减余额/可领取数量、消耗一次性状态并更新相关总量，再转账。赎回先销毁/扣减份额再付款。
- ERC20 使用 `safeTransfer`，ERC721 使用要求的安全转账；原生资产使用检查成功结果的 value call 或适当的已安装 OpenZeppelin 工具。转账失败必须回退整个操作，恢复会计状态。
- ERC721 safe mint 和转账属于外部交互：接收方回调前先完成相应业务状态。不吞掉转账错误，不在失败后记录成功。

复合操作明确组合这些阶段：流入资产先到账再记入，流出资产先完成扣账再发送。普通存取款请求不引入提前授信/闪电贷流程。

## 强制执行合法调用流程

从用户要求与现有代码推导合法状态、参与者、前置条件、转换和后置条件，用实际校验执行，不只写注释。非预期调用者、不支持的状态转换、重复领取、顺序错误、过期输入、回调不匹配及外部操作失败，必须通过适当错误原子回退；保留有用的依赖错误，不吞掉异常。

除非部分成功/恢复是明确定义的功能，否则不静默跳过失败批次项、不返回看似成功的默认值、不重试另一条业务路径、不 catch 后继续。按适用范围拒绝不支持的 fallback 调用和普通的意外原生资产转入；仅让 `receive` 回退无法阻止强制转入原生资产。

预期前置条件使用 `require` 搭配 custom error；没有合法后续路径的分支可使用 `revert CustomError(...)`。`assert` 仅用于真正的内部不变量，不替代不可信入参校验。

## 验证与完成

使用项目实际配置编译。补充并运行聚焦测试：入参错误、越权、暂停/恢复与紧急策略、流入到账前不记账、流出回调前已扣账、到账不符、转账失败后的回滚、含跨入口回调的重入，以及非法/重复状态转换。按需补充 token 标准及 UUPS 初始化、权限和状态保持测试。用能在回调中观察状态的 mock 验证执行顺序，不只检查最终余额。

同步受影响 caller、部署脚本、错误断言、接口和文档。运行修改文件的格式检查并审阅 diff，逐项核对构造/初始化参数尾下划线、Ownable 与调用者 owner、private constant/immutable、所有 private 状态变量前下划线、普通配置的读取需求、共享配置 helper、业务入口与相关 helper 是否就近成组，以及地址校验未加入代码长度限制。共享会计、继承或初始化修改覆盖全部受影响套件；影响面无法限定，或用户/项目要求时运行全量。必需检查通过后，仅因新改动、失败或未解决疑点重复验证。

简短报告已实现行为、实际验证和剩余阻塞。无证据时不宣称执行成功或升级兼容。
