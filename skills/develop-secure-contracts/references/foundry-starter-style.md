# foundry-starter 编码与部署约定

用户以 foundry-starter 为开发起点。目标是让新增 `src/` 和 `script/` 代码自然融入现有项目，避免完成后再重写结构。以当前项目代码为事实来源，以下约定用于识别和延续模板风格，不替代业务要求与安全检查。

## 开发前读取

先读当前项目的这些文件或其等价文件：

- `src/Wallet.sol`：导入、继承、命名、错误、函数和事件组织。
- `src/lib/Errors.sol`：共享错误定义与使用方式。
- `script/Wallet.s.sol`：可执行部署入口与广播边界。
- `script/ContractCollection.sol`：部署、初始化、配置及升级 helper。
- `script/BaseDeployScript.sol`：链配置与注册表访问封装。
- `script/ContractRegistry.sol`：地址记录格式与读写规则。
- `foundry.toml`：编译器、remapping、格式化和脚本配置。

项目已替换 Wallet 时，读取最接近的新合约及其部署入口。模板文件缺失时，可参考用户提供的 foundry-starter；本机已知位置为 `/Users/derick/Documents/git-project/contracts/evm/foundry-starter`，不可将该绝对路径作为技能运行前提。其他环境无法读取时沿用本文件与项目现有约定，不擅自下载或覆盖模板。

参考提取自 2026-10-19 本地 foundry-starter，commit `c07660104dd10788777871965d5c7848ed983570`。执行时优先读取当前项目，不假设模板版本、依赖或初始化签名永久不变。

## src/ 的风格

- 延续文件的 SPDX、pragma 和实际依赖路径，不为了模仿模板升级编译器。使用具名导入；按依赖组件、工具、项目自有库分组，保持周边顺序。
- 多继承声明按现有样式分行。先列 `using`、常量/immutable、状态变量与事件，再组织构造、初始化和相关函数。相关 struct、modifier、内部 helper 靠近使用它们的逻辑；不要为全局统一排序移动已有状态变量。
- 函数按业务流程与调用关系分组：入口后紧跟专用 internal/private helper，再进入下一组业务函数；共享 helper 放在主要入口之后或所属业务组末尾，配置入口与配置 helper 相邻。遵循技能入口“按业务流程组织函数”的规则，不按可见性将内部函数统一放到文件末尾。
- 默认复用 `src/lib/Errors.sol`，通过 `Errors.InvalidParameter()` 等命名空间引用错误。语义匹配时复用现有错误，需要新业务错误时补入该库；不为每个参数机械新增一个错误。脚本注册表已有局部错误保持局部，不为集中化改写它。
- 入口校验使用 `require(condition, Errors.ErrorName())`。同一入口、同一阶段且错误语义相同的多个地址非零校验默认用 `&&` 合并为一条 `require`，保留原错误 selector 与失败参数（零地址错误可用 `Errors.InvalidAddress(address(0))`），不逐个地址重复写，也不为合并引入三元表达式或新错误；权限、资产互异、资产支持、签名等不同失败原因保持独立校验。具体合法输入由业务决定，不复制模板的限制。
- 状态变量和普通参数采用现有 camelCase；`constructor` 和 `initialize` 的所有具名参数均使用尾下划线，如 `tokenRecipient_`、`signer_`、`amount_`，不以是否与状态同名为条件。内部 helper 沿用 `_updateTokenAllowed`、`_onlyEOA` 等前下划线。所有 constant/immutable 状态变量均不加前导下划线：constant 使用 `UPPER_SNAKE_CASE`，immutable 使用 `camelCase`（例如 `MAX_AMOUNT`、`token`）；仅普通 private 状态变量使用 `_camelCase`，已有 `__gap` 等升级保留字段沿用原名。constant/immutable 默认声明为 `private`；由 `keccak256(...)` 计算的常量声明为 `public constant`（例如 `bytes32 public constant ORDER_TYPEHASH = keccak256("Order(...)");`），通过自动 getter 查询具体值，不再额外包装同义 getter。需要外部核对的普通配置使用 `public`。
- 配置入口默认沿用 `updateXxx`、`UpdateXxx` 事件的命名习惯。已有 API 名称保持不变，不为风格重命名。写入状态后按项目约定发出事件。
- 权限使用 `Ownable`（代理用 `OwnableUpgradeable`），构造/初始化不接收 owner 参数，直接以 `msg.sender` 初始化 owner；不主动引入两步所有权或角色权限体系。
- 地址有效性 `require` 只检查非零，不添加 `code.length`、`extcodesize` 或 `isContract` 限制；零地址有明确业务语义时保留。
- 构造/初始化设置且外部入口仍可修改的配置，必须共用 `_setXxx` 或 `_updateXxx` internal helper，集中校验、赋值与配置事件；初始化与外部 setter 均调用它，权限与 guard 留在外部入口。仅初始化写入且不对外提供修改入口的变量可直接赋值。
- 仅在一个地方调用且只有一两条简单语句的 internal/private helper，直接内联到调用方，例如“读取 storage 引用 + 一条 require”；按实际语句判断，不按换行或注释行数判断。重复校验或复用业务步骤再抽内部 helper；单次调用的复杂算法或独立完整业务阶段可按可读性保留。初始化与外部配置入口共用的 helper 仍按上述要求保留。需要 modifier 时可沿用薄 modifier 调用内部校验函数的写法，不为单次调用拆出无意义的层级、配置框架或基类。
- 保持 Wallet 风格的紧凑 NatSpec，说明实际行为、参数、单位及重要限制，不照抄错误或缺失的注释。注释语言沿用目标文件或用户要求，不因为技能正文是中文而批量翻译项目注释。
- 遵循目标项目格式化配置，保留无关代码；不把模板里的尾随空格、不规则空行等偶然排版当成必需风格。

## script/ 的职责分工

| 文件 | 应承担的职责 | 不应混入的职责 |
| --- | --- | --- |
| `<Name>.s.sol` | 继承 `ContractCollection`，读取部署身份，`run()` 中控制广播、获取配置并调用部署 helper | 重复实现代理部署、JSON 注册表或另一套配置体系 |
| `ContractCollection.sol` | 接收明确参数，创建合约，完成对应初始化/必要配置，调用 `saveContract`，返回具体合约类型 | 读取私钥/环境变量，调用 `startBroadcast` / `stopBroadcast` |
| `BaseDeployScript.sol` | 复用 `Config`、`_getConfig()`、`getContractAddress()`、`saveContract()` 及可覆盖的注册表文件名 | 为单个新合约重复实现底层 JSON 读写 |
| `ContractRegistry.sol` | 维护按 chain ID 和合约键组织的地址记录及必要校验 | 具体业务部署编排 |

优先扩展现有文件和 `Config`，不要为每个合约重新创建上述四层。当前项目已有不同配置实现时保留其接口。仅用户要求新增部署入口时创建入口，不为纯合约逻辑修改自动新增脚本。

### 部署 helper

按 `deployWallet` 的结构组织一个完整 helper：

1. 使用明确、具名的业务参数，不要求上层传入不透明的 `bytes initData`。
2. 创建实现，使用项目现有 key 保存实现地址，例如 `walletImpl`。
3. 在 helper 内使用 `abi.encodeCall(Wallet.initialize, (tokenRecipient, signer))` 构造初始化数据；按实际签名核对顺序，不使用字符串形式的 `abi.encodeWithSignature`。
4. 创建 `ERC1967Proxy` 并原子初始化；返回 `Wallet wallet` 这样的具体类型并保存代理地址，例如 `wallet`。
5. 完成该部署所需且已授权的业务配置，如支持资产设置。验证执行身份与初始化得到的权限一致；owner 取实际构造/初始化调用者 `msg.sender`，工厂部署需核对工厂是否成为 owner，不额外传入 owner 参数。

已有 helper 是完整部署单元时，不无故拆成上层需要手动组合的“实现部署”和“代理部署”两个函数。普通非代理合约直接部署并保存，不强行添加 UUPS。新增 key 沿用项目的 lowerCamelCase，保留既有 key 和注册表 schema。

入口沿用项目 `PRIVATEKEY` 等现有环境变量名，只在外层持有广播控制。配置来自 `BaseDeployScript` 的实际入口；不硬编码复制模板中的链 ID、收款地址、签名者或代币地址。对未支持链及缺失必需配置在产生交易前明确失败，不以默认零值继续部署。

第三方协议地址按目标链及版本纳入现有配置，连接真实部署。`ContractCollection` 不部署协议 mock 或仿制品；需要新池时通过真实协议的 Factory/管理器创建。具体规则见技能入口的“第三方协议使用目标网络真实部署”。

### 升级 helper

复用当前链注册表获取代理，创建新实现，将**新实现地址**传入代理的 `upgradeToAndCall`，迁移 calldata 按实际函数使用类型化编码。注册表写入遵循项目约定，但注册表中的新地址不等于升级已经成功；验证必须核对实际实现指向和状态保持。

参考快照中的 `upgradeWallet()` 将 `walletAddr` 自身传为升级目标，而不是新建的 `walletImpl`。这是不能复制的逻辑问题，不属于编码风格。提取模板模式时必须检查调用参数与目标，不将模板当作已验证实现。

## 风格与业务边界

Wallet 的 EOA 限制、签名提款、nonce、资产转发到收款人、allowlist 和具体链配置都是示例业务，不自动推广到其他合约。是否升级及具体资金流由请求决定；需要重入保护时直接使用 `ReentrancyGuardTransient`，并按技能入口核对已安装依赖、编译配置与目标链的 EIP-1153 支持。模板中的其他重入锁写法不覆盖此规则。

保留用户明确要求的资金流与校验规则。模板行为与要求冲突时指出具体差异，选择满足要求且延续结构的实现，不以“保持风格”为由复制缺陷或省略安全保证。

## 交付前对照

在正常编译和行为验证之外，对照当前参考检查：共享错误是否复用、命名是否一致、helper 是否过度拆分、广播是否仅在入口、配置是否集中、初始化编码是否类型化、实现/代理 key 是否分别正确、升级目标是否为新实现。只检查本次修改范围，不借机格式化或重构整个模板。
