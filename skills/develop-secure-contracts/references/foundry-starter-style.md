# foundry-starter 编码与部署约定

以当前项目的等价文件为依据；模板仅用于延续组织风格，不复制其业务限制、固定地址或缺陷。通用编码规则见技能主文件。

## 开发前读取

读取 `foundry.toml`、`src/Wallet.sol`、`src/lib/Errors.sol` 及相关部署文件；Wallet 已替换时选最接近的业务合约。涉及部署时读取 `script/Wallet.s.sol`、`ContractCollection.sol`、`BaseDeployScript.sol`、`ContractRegistry.sol` 的等价文件。文件缺失时沿用现有结构，不擅自下载或覆盖模板。

## 源码风格

- 延续 SPDX、pragma、实际依赖路径和多继承排版；使用具名导入，按依赖组件、工具、项目自有库分组，保持周边顺序。
- 先列 using、常量/immutable、状态变量与事件，相关 struct/modifier 靠近所属逻辑；函数组织按主文件，不移动已有存储变量来统一排序。
- 配置入口/事件默认沿用 `updateXxx` / `UpdateXxx`，状态写入后发事件；保留已有 API 名称。
- NatSpec 紧凑说明实际行为、参数、单位及限制，语言沿用目标文件或用户要求。遵循项目 formatter，不复制模板偶然排版。

## 部署职责

| 文件 | 职责 |
| --- | --- |
| `<Name>.s.sol` | 继承 ContractCollection，读取部署身份，在 run 中控制广播、获取配置并调用 helper。 |
| `ContractCollection.sol` | 接收明确参数，部署、初始化及必要配置，保存地址并返回具体类型；不读取私钥或控制广播。 |
| `BaseDeployScript.sol` | 管理链配置，复用注册表访问与可覆盖的注册表文件名。 |
| `ContractRegistry.sol` | 按 chain ID 和合约键维护地址及校验，不编排业务部署。 |

扩展现有文件与 Config，不重复建立这四层；项目已有不同实现时沿用其接口。仅用户要求时新增部署入口。广播仅由外层控制，沿用现有私钥环境变量；配置不硬编码，缺失必需项或链不受支持时在产生交易前失败。

## 部署与升级 helper

- 使用明确、具名的业务参数，不要求上层传不透明 initData。代理部署在同一 helper 内创建实现、以 `abi.encodeCall` 编码初始化、创建 ERC1967Proxy、完成必要配置并保存地址；不无故拆成需上层手动组合的步骤。
- 返回具体合约类型，按现有 key 区分实现与代理，如 walletImpl / wallet；新增 key 用 lowerCamelCase，保留注册表 schema。普通非代理合约直接部署，不强加 UUPS。
- 协议地址和市场参数纳入现有链配置；真实部署与 Fork 验证遵循主文件。
- 升级从当前链注册表读取代理，创建新实现，将新实现地址传入代理 `upgradeToAndCall`，迁移参数用类型化编码。保存新地址不代表升级成功，须核对实现指向与状态保持。
