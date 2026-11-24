# 合约编码与部署约定

本文件提供可直接用于新项目的默认编码与部署约定。新项目直接采用本文件和技能主文件的规则；已有项目保留 API、存储和明确项目约定。

## 开发前读取

已有项目读取 `foundry.toml`、`src/lib/Errors.sol`、本次相关业务合约和部署文件。涉及部署时读取 `<Name>.s.sol`、`ContractCollection.sol`、`BaseDeployScript.sol`、`ContractRegistry.sol` 的现有等价文件。新项目不存在这些文件时按本技能创建任务需要的文件，不要求查找其他项目的示例合约，不擅自下载或覆盖模板。

## 新项目默认值

- 使用 Foundry 的 `src/`、`src/interfaces/`、`src/lib/`、`script/`、`test/` 目录，按需求创建；业务共用错误放在 `src/lib/Errors.sol`。
- 新建自有 Solidity 文件默认使用 `// SPDX-License-Identifier: MIT`，pragma 与项目选定且支持所需功能的编译器一致，使用 Foundry formatter。
- 使用具名导入；OpenZeppelin 依赖在前，空行后列项目接口及库。标识符使用英文，NatSpec 和必要业务注释默认中文。
- 合约内按 using、常量/immutable、相关 struct/enum 与状态、事件、modifier、构造/初始化、业务函数组织；业务入口与其 helper 相邻，具体命名、权限及校验遵循技能主文件。
- 新建配置 API 时依赖装配使用 `configureXxx` / `ConfigureXxx`，运行期参数更新使用 `updateXxx` / `UpdateXxx`；初始化共用 `_setXxx` / `_updateXxx`。不需要查阅旧项目才能决定这些写法。

## 源码风格

- 延续 SPDX、pragma、实际依赖路径和多继承排版；使用具名导入，按依赖组件、工具、项目自有库分组，保持周边顺序。
- 先列 using、常量/immutable、状态变量与事件，相关 struct/modifier 靠近所属逻辑；函数组织按主文件，不移动已有存储变量来统一排序。
- 配置入口/事件沿用同一合约同类业务的命名：`configureXxx` / `ConfigureXxx` 或 `updateXxx` / `UpdateXxx`，状态写入后发事件。已有两种命名并存时按所属业务组选择，不为统一风格重命名 API；缺少先例时，依赖装配用 configure，运行期参数更新用 update。
- NatSpec 紧凑说明实际行为、参数、单位及限制，语言沿用目标文件或用户要求。遵循项目 formatter，不复制模板偶然排版。
- 中文项目以中文描述业务；公共入口、事件、struct 及重要字段写清资产归属、单位、边界和状态含义。初始化成组配置可用简短中文注释解释精度、规模和用途，避免逐行复述赋值。
- 比例和精度计算优先复用已安装的 `Math.mulDiv`、`SafeCast`；写清向下/向上取整的业务方向与余数归属，不为缩短代码改用存在中间溢出或截断风险的运算。数字分隔及短 if 排版跟随项目 formatter。

## 部署职责

| 文件 | 职责 |
| --- | --- |
| `<Name>.s.sol` | 继承 ContractCollection，读取部署身份，在 run 中控制广播、获取配置并调用 helper。 |
| `ContractCollection.sol` | 接收明确参数，部署、初始化及必要配置，保存地址并返回具体类型；不读取私钥或控制广播。 |
| `BaseDeployScript.sol` | 管理链配置，复用注册表访问与可覆盖的注册表文件名。 |
| `ContractRegistry.sol` | 按 chain ID 和合约键维护地址及校验，不编排业务部署。 |

扩展现有文件与 Config，不重复建立这四层；项目已有不同实现时沿用其接口。仅用户要求时新增部署入口。广播仅由外层控制，沿用现有私钥环境变量；配置不硬编码。遵循技能主文件的脚本校验约定。

## 部署与升级 helper

- 使用明确、具名的业务参数，不要求上层传不透明 initData。代理部署在同一 helper 内创建实现、以 `abi.encodeCall` 编码初始化、创建 ERC1967Proxy、完成必要配置并保存地址；不无故拆成需上层手动组合的步骤。
- 返回具体合约类型，按现有 key 区分实现与代理，如 `<name>Impl` / `<name>`；新增 key 用 lowerCamelCase，保留注册表 schema。普通非代理合约直接部署，不强加 UUPS。
- 协议地址和市场参数纳入现有链配置；真实部署与 Fork 验证遵循主文件。
- 升级从当前链注册表读取代理，创建新实现，将新实现地址传入代理 `upgradeToAndCall`，迁移参数用类型化编码。保存新地址不代表升级成功，须核对实现指向与状态保持。
