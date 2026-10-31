---
name: develop-secure-contracts
description: "用于通过 OpenZeppelin 权限、重入保护、紧急暂停、ERC20、ERC721 或 UUPS 升级组件开发 Solidity 合约，严格执行入参校验、资金流顺序与合法调用流程。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# 使用 OpenZeppelin 开发合约

## 执行与兼容性

完成用户授权的实现；用户指令优先。编辑前读取项目配置、目标代码、相关测试/脚本及实际安装依赖。保留无关改动，不为统一风格改写范围外代码。

确认实际编译器/backend 支持 `require(condition, CustomError(...))`；不静默升级或改成 revert string、`if/revert`。条件不足时报告具体阻塞，继续独立工作。

保持已有函数/event ABI、错误约定、权限、资产语义及升级存储兼容性；授权变更时同步受影响依赖。EIP-712 字段名和类型字符串参与签名摘要；存储标识重命名也需核对升级工具，不能仅凭槽位未变认定兼容。

## 项目风格与错误库

默认沿用 foundry-starter 风格；编写 `src/`、`script/` 前读取[编码与部署约定](references/foundry-starter-style.md)，对照当前项目等价文件。模板业务限制、地址和逻辑不得机械复制，旧写法不覆盖本技能的新增代码约定。

`src/lib/Errors.sol` 仅保留 `src` 合约实际使用的 error。

## 函数组织

- 仅调用一次且只有一两条简单语句的 internal/private helper 直接内联，按实际复杂度判断，不按换行或注释行数判断。复用逻辑、复杂算法或完整业务阶段可保留 helper；初始化与配置入口共用的 helper 必须保留。
- 按“业务入口 → 专用 helper → 下一组业务函数”组织，配置入口与配置 helper 相邻，多层 helper 按调用链排列；不按可见性把内部函数统一堆在末尾。
- 共享 helper 只保留一份，放在主要入口之后或所属业务组末尾。移动函数时连同注释移动，保持逻辑、modifier、继承和存储布局。

## 命名、权限与配置

- 名称体现准确的业务概念，简洁、利用上下文去重，同一概念使用一致术语；保留区分资产、归属和状态的必要限定，不直译整条业务规则、不强套术语或使用含糊缩写。变量用名词/状态短语，布尔值用谓词，函数用业务动作。
- 区分本金与奖励、资产与份额、请求量与实收量、累计/已领/可领；`pending` 说明等待何种状态，`vested` 仅用于归属机制。基点用 `Bps` / `_BPS`，区分比例、速率、时间戳和时长；NatSpec 说明单位、精度、分母及重要限制。
- 普通变量和参数用 camelCase；`constructor`、`initialize` 的所有具名参数统一加尾下划线。internal/private helper 用前下划线。
- constant 用 `UPPER_SNAKE_CASE`，immutable 用 camelCase，两者均不加前下划线、默认 private；`keccak256(...)` 常量用 public constant 并直接通过 getter 暴露。其他 private constant/immutable 仅在业务需要时提供显式 getter。仅普通 private 状态变量用 `_camelCase`；需要外部核对的普通配置用 public，保留已有升级保留字段。
- 权限用 `Ownable`，代理用 `OwnableUpgradeable`；不主动引入两步所有权或 AccessControl。按安装版本通过 `Ownable(msg.sender)` / `__Ownable_init(msg.sender)` 设置 owner，不接收 owner/admin 参数，不用 `tx.origin`；工厂部署须核对真实调用者。
- 构造/初始化与外部配置入口写入同一可变配置时，共用 internal `_setXxx` / `_updateXxx`，集中校验、赋值和事件；权限及适用 guard 留在外部入口，不通过 `this` 调用 setter。仅初始化写入的配置可直接赋值。
- 地址有效性只检查非零，不加 `code.length`、`extcodesize`、`isContract` 限制；保留零地址的明确业务语义，权限、支持资产及协议身份另行核对。

## 第三方协议

- 集成目标链真实部署的协议，核对 chain ID、版本、官方地址及市场/池的链上配置；使用匹配接口或官方依赖。adapter、策略和回调必须调用真实协议，不以本地仿制品替代。
- 协议地址和市场参数纳入现有链配置；需要新池/头寸时通过真实协议入口创建。
- 在目标网络固定区块 Fork 上验证真实调用，遵循 `foundry-test`；mock 通过不能替代集成验证，Fork 通过不代表已广播。
- 缺少协议部署、RPC、历史状态或配置时报告缺口，不静默换链、换版本。OpenZeppelin 等源码组件仍直接导入或继承。

## OpenZeppelin 组件

按实际需求选用安装版本的组件，核对 API、hook、构造和初始化方式，不复制源码或机械叠加组件。

- 重入：资产转移入口及共享暴露会计状态的其他可写入口，普通/代理合约均用 `ReentrancyGuardTransient` 和 `nonReentrant`。确认依赖、编译配置及目标链支持 EIP-1153；不静默替换或升级依赖。受保护入口复用 internal helper，避免嵌套进入锁。
- 暂停：需要时用 `Pausable` 和有权限的暂停/恢复入口，明确暂停范围与紧急退出，并实际应用 guard/hook。
- ERC20：继承标准组件及必要扩展，外部代币用 `SafeERC20`，保留标准零额转账、allowance、mint/burn 和事件语义。
- ERC721：继承标准组件及必要扩展，将 safe transfer/mint 作为外部回调边界。
- UUPS：使用安装版本的 UUPS 模式与 `ERC1967Proxy`，通过 `_authorizeUpgrade` 限权；代理部署原子初始化，必需父级按顺序各初始化一次，实现合约禁用 initializer。Reinitializer 仅用于明确迁移。

供已有代理使用的新实现即使未部署也属于升级。验证继承、storage、namespaced storage 和升级授权；宣称布局兼容前对照已部署基线或运行升级验证器。

## 入口校验与错误

每个函数在入口用 `require(condition, CustomError(...))` 建立适用前置条件，位于业务写入和资产交互之前；可通过 modifier/helper 执行。所有可达 caller 已保证且使用前不会失效的条件不重复检查。

入参合法性统一使用无参 `Errors.InvalidParameter()`，不按参数分别定义错误或传入失败值。权限、暂停、业务状态、签名、重入、实际到账及外部调用失败保留专用错误和依赖错误。

同一入口、同一阶段按校验含义分组：金额正数、地址非零、时间、金额上限/余额等分别成组；同类用 `&&` 合并，不同类分开，即使 Solidity 类型相同。保持数组边界、非零除数和算术范围的安全求值顺序，不跨状态写入或外部交互合并。

```solidity
require(amount > 0 && minimumOutput > 0, Errors.InvalidParameter());
require(receiver != address(0) && token != address(0), Errors.InvalidParameter());
```

有效输入由业务决定，保留合法零额、token ID 零等情况。依赖状态使用权威值，只有交互后可知的条件在交互后检查。错误参数须低成本、无副作用，因为 `require` 成功时也会求值。无合法后续路径可 `revert CustomError(...)`，`assert` 仅用于内部不变量。

## 资金流顺序

外部交互前进入重入保护；重入锁和必要的最小回调认证状态不属于业务入账。不得向回调或敏感 view 暴露可被利用的资产/会计中间快照。

### 流入：校验 → 收到资产 → 验证到账 → 修改业务状态

- 先校验参数、权限、暂停及合法状态；ERC20 用 `safeTransferFrom`。到账验证前不增加余额、份额、存款或奖励。
- 足额到账要求余额增量等于请求量；明确支持转账税时按实收量记账并检查最低到账要求，不按未经验证的请求量记账。
- ERC721 认证预期 token、operator/from/token ID 和回调上下文，转账后确认所有权。必需接收回调只确认预期转账，不入账、不开放业务入口；若 `nonReentrant` 阻止正常接收，改用回调认证，拒绝非预期/重复回调。
- 原生资产先验证 `msg.value` 再记账；直接捐赠或强制转入不得自动计入用户资产。验证后统一更新会计并发出成功事件。

### 流出：校验 → 修改业务状态 → 转出资产

- 校验接收者、权益、流动性及合法状态；先扣减权益/份额、消耗一次性资格并更新总量，再转出。
- ERC20 用 `safeTransfer`，ERC721 用所需安全转账；原生资产检查 value call 成功或使用适用 OpenZeppelin 工具。safe mint 同样先完成状态，再触发回调。
- 转账失败原子回退、恢复会计，不吞掉异常或记录成功。复合操作分别遵守流入/流出顺序；普通存取款不引入提前授信或闪电贷。

## 合法调用流程

按业务实现参与者、状态、顺序和前后置条件的实际校验；拒绝越权、不支持的状态转换、重复领取、过期输入和回调不匹配。除非明确设计部分成功/恢复，否则失败批次不跳过、不返回成功默认值、不重试其他路径或 catch 后继续。

按业务拒绝不支持的 fallback 和意外原生资产转入；`receive` 回退不能阻止强制转入。

## 验证与完成

使用项目配置编译、检查修改文件格式并审阅 diff，核对上述约定；同步受影响 caller、脚本、错误断言、接口和文档。

补充并运行受影响测试，覆盖适用的入参、权限、暂停、到账、会计顺序、失败回滚、跨入口重入、非法/重复状态转换及 token/UUPS 行为。用可观察回调状态的测试探针验证执行顺序；第三方集成仍须真实 Fork。共享会计、继承或初始化修改覆盖所有受影响套件，影响面无法限定或用户/项目要求时运行全量。

必需检查通过后仅因新改动、失败或未解决疑点重复验证。报告实际改动、验证结果和剩余阻塞，无证据不宣称执行成功或升级兼容。
