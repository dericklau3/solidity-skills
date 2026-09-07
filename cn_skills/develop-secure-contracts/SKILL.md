---
name: develop-secure-contracts
description: "使用 OpenZeppelin Contracts 库开发安全的智能合约。当用户需要将 OpenZeppelin 库组件集成到现有或新的合约中时使用，包括代币标准（ERC20、ERC721、ERC1155）、访问控制（Ownable、AccessControl、AccessManager）、安全组件（Pausable、ReentrancyGuard）、治理（Governor、timelock）或账户（多签、账户抽象）等。涵盖从库源码中发现集成模式、使用 CLI 合约生成器，以及库优先（library-first）的集成方法。支持 Solidity、Cairo、Stylus、Stellar 和 Sui Move。"
license: AGPL-3.0-only
metadata:
  author: OpenZeppelin
---

# 使用 OpenZeppelin 开发安全的智能合约

## 核心工作流程

### 回复前先理解用户需求

对于概念性问题（例如“Ownable 是如何工作的？”），只进行解释，不需要生成代码。对于实现类请求，则按照下面的工作流程执行。

### 关键要求：始终先阅读项目

在生成代码或建议修改之前：

1. **搜索用户项目**中的现有合约（使用 `Glob` 搜索 `**/*.sol`、`**/*.cairo`、`**/*.rs`、`**/*.move` 等）
2. **阅读相关合约文件**，了解项目中已经存在的实现
3. **默认采用集成，而不是替换** —— 当用户说“增加暂停功能”或“改成可升级合约”时，默认含义是修改现有代码，而不是重新生成一个全新的合约。只有用户明确要求（例如“从头开始”“替换这个合约”）时才进行替换。

如果某个文件无法读取，必须明确指出失败情况 —— 报告尝试读取的路径以及失败原因，并询问路径是否正确。绝不能在文件实际上无法读取时，静默退回到通用回答，好像该文件不存在一样。

### 基本原则：优先使用库组件，而不是自定义代码

在编写**任何**逻辑之前，先搜索 OpenZeppelin 库中是否已经存在对应组件：

1. **存在完全匹配的组件？** 直接导入并使用 —— 继承它、实现它的 trait，或通过组合方式使用。完成。
2. **存在接近需求的组件？** 导入并扩展它 —— 只重写库明确允许重写的函数（例如 `virtual` 函数、hook、可配置参数）。
3. **完全没有匹配组件？** 只有这种情况下才编写自定义逻辑。在此之前，必须先浏览库的目录结构进行确认。

**绝不要把库的源码复制或嵌入到用户合约中。** 应始终从依赖中进行 import，这样项目才能持续获得安全更新。库中已经提供的功能，不要手写重复实现：
- 当已经存在 `Pausable` 或 `ERC20Pausable` 时，不要自己编写 `paused` modifier
- 当已经存在 `Ownable` 时，不要自己编写 `require(msg.sender == owner)`
- 当库的基础合约已经处理 ERC165 时，不要自行实现 ERC165 逻辑

### 方法论

主要工作流程是：**通过阅读库源码发现集成模式**。

1. 检查用户项目当前已经 import 了哪些内容
2. 阅读项目已安装依赖中的源码和文档
3. 确定该依赖要求哪些函数、modifier、hook 和 storage
4. 将这些要求应用到用户的合约中

完整的逐步流程见下方 [模式发现与集成](#模式发现与集成)。

### 将 CLI 生成器作为参考

使用 `npx @openzeppelin/contracts-cli` 生成参考实现，用于发现正确的集成模式：
先生成一个基础版本到文件，再生成一个启用目标功能的版本到另一个文件，对两者执行 diff，然后把差异应用到用户代码中。CLI 输出应视为规范的正确集成参考 —— 用它来确认某个功能需要哪些 import、继承、storage 和 override。

关于“生成 → 比较 → 应用”的详细流程，请参见 [CLI 生成器](#cli-生成器)。

如果需要的功能没有对应 CLI 命令，则使用 [模式发现与集成](#模式发现与集成) 中的通用方法。没有 CLI 命令并不代表库不支持该功能，只代表没有对应生成器。

## 模式发现与集成

这是一个通过阅读依赖源码，发现并应用 OpenZeppelin 合约集成模式的流程指南。适用于任何生态系统和任何库版本。

**前置条件：** 始终遵循上面的“库优先”决策树（优先使用库组件而不是自定义代码，并且绝不复制/嵌入库源码）。

### 第 1 步：识别依赖并搜索库

1. 搜索项目中的合约文件：使用 `Glob` 搜索 `**/*.sol`、`**/*.cairo`、`**/*.rs`、`**/*.move`，或使用下方查询表中对应生态的文件扩展名。
2. 阅读现有合约中的 import/use 语句，确认当前已经使用了哪些 OpenZeppelin 组件。
3. 在项目依赖树中找到已经安装的依赖：
   - Solidity：`node_modules/@openzeppelin/contracts/`（Hardhat/npm）或 `lib/openzeppelin-contracts/`（Foundry/forge）
   - Cairo：根据 `Scarb.toml` 中的依赖定位 —— 源码由 Scarb 缓存
   - Stylus：根据 `Cargo.toml` 定位 —— 源码位于 `target/` 或 Cargo registry 缓存（`~/.cargo/registry/src/`）
   - Stellar：根据 `Cargo.toml` 定位 —— 使用与 Stylus 相同的 Cargo 缓存位置
   - Sui Move：根据 `Move.toml` 定位 —— 构建后，MVR 源码缓存于 `~/.move/`，同时每个依赖会镜像到 `build/<project_package>/sources/dependencies/<move_package_name>/`
4. 浏览依赖目录，发现可用组件。对已安装源码使用 `Glob` 模式搜索（例如 `node_modules/@openzeppelin/contracts/**/*.sol`）。不要凭记忆假设库里有哪些内容 —— 必须通过列出目录进行验证。
5. 如果本地没有安装该依赖，则 clone 或浏览官方仓库（见下方查询表）。

### 第 2 步：阅读依赖源码和文档

1. 阅读与用户需求相关组件的源码文件。
2. 查找源码中的文档：Solidity 中的 NatSpec 注释（`///`、`/** */`），Rust 和 Cairo 中的文档注释（`///`），以及组件目录中的 README 文件。
3. 使用“基本原则”中的决策树确定集成策略：
   - 如果组件可以直接满足需求 → import 并原样使用。
   - 如果需要定制 → 找出库提供的扩展点（`virtual` 函数、hook 函数、可配置的构造函数参数），然后 import 并扩展。
   - 只有当没有任何组件覆盖需求时 → 才编写自定义逻辑。
4. 确定**公共 API**：暴露的函数/方法、发出的事件、定义的错误。
5. 确定**集成要求** —— 这是最关键的一步：
   - 集成方**必须**实现的函数（抽象函数、trait 方法、hook）
   - 必须应用到集成方函数上的 modifier、decorator 或 guard
   - 必须传入的 constructor 或 initializer 参数
   - 必须声明的 storage 变量或状态
   - 必须实现的继承关系或 trait（始终通过 import 完成，绝不能复制源码）
6. 在同一仓库中搜索示例合约或测试，确认正确用法。优先检查 `test/`、`tests/`、`examples/` 或 `mocks/` 目录。

### 第 3 步：提取最小集成模式

根据第 2 步的结果，构建实现该功能所需的最小修改集合：

- 需要新增的 **import / use 语句**
- 需要新增的**继承 / trait 实现**（始终从依赖中 import）
- 需要声明的 **storage**
- **constructor / initializer** 修改（新增参数、初始化调用）
- 需要新增的**函数**（必须实现的 override、hook、公共 API）
- 需要修改的**现有函数**（增加 modifier、调用 hook、触发 event）

如果合约是可升级合约，上述任何改动都可能影响 storage 兼容性。在应用修改之前，应先查阅对应的升级类 skill。

不要加入依赖本身并不要求的额外内容。这应该是“没有该功能的合约”和“具有该功能的合约”之间的**最小 diff**。

### 第 4 步：将模式应用到用户合约

1. 阅读用户现有的合约文件。
2. 使用 `Edit` 工具应用第 3 步中的修改。不要替换整个文件 —— 应将改动集成到现有代码中。
3. 检查冲突：重复的访问控制系统、冲突的函数 override、不兼容的继承关系。完成前必须解决这些问题。
4. 不要让用户自己去修改 —— 应直接应用修改。

### 仓库与文档查询表

| 生态 | 仓库 | 文档 | 文件扩展名 | 依赖位置 |
|-----------|-----------|---------------|----------------|-------------------|
| Solidity | [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) | [docs.openzeppelin.com/contracts](https://docs.openzeppelin.com/contracts) | `.sol` | `node_modules/@openzeppelin/contracts/` 或 `lib/openzeppelin-contracts/` |
| Cairo | [cairo-contracts](https://github.com/OpenZeppelin/cairo-contracts) | [docs.openzeppelin.com/contracts-cairo](https://docs.openzeppelin.com/contracts-cairo) | `.cairo` | Scarb 缓存（根据 `Scarb.toml` 定位） |
| Stylus | [rust-contracts-stylus](https://github.com/OpenZeppelin/rust-contracts-stylus) | [docs.openzeppelin.com/contracts-stylus](https://docs.openzeppelin.com/contracts-stylus) | `.rs` | Cargo 缓存（`~/.cargo/registry/src/`） |
| Stellar | [stellar-contracts](https://github.com/OpenZeppelin/stellar-contracts)（[Architecture](https://github.com/OpenZeppelin/stellar-contracts/blob/main/Architecture.md)） | [docs.openzeppelin.com/stellar-contracts](https://docs.openzeppelin.com/stellar-contracts) | `.rs` | Cargo 缓存（`~/.cargo/registry/src/`） |
| Sui Move | [contracts-sui](https://github.com/OpenZeppelin/contracts-sui)（[llms.txt](https://raw.githubusercontent.com/OpenZeppelin/contracts-sui/main/llms.txt) · [ARCHITECTURE](https://raw.githubusercontent.com/OpenZeppelin/contracts-sui/main/ARCHITECTURE.md)） | [docs.openzeppelin.com/contracts-sui](https://docs.openzeppelin.com/contracts-sui) | `.move` | Move Registry 缓存（`~/.move/`，根据 `Move.toml` 定位） |

### 目录结构约定

在各个仓库中，通常可以从以下位置寻找对应组件：

| 类别 | Solidity | Cairo | Stylus | Stellar |
|----------|---------|-------|--------|---------|
| 代币 | `contracts/token/{ERC20,ERC721,ERC1155}/` | `packages/token/` | `contracts/src/token/` | `packages/tokens/` |
| 访问控制 | `contracts/access/` | `packages/access/` | `contracts/src/access/` | `packages/access/` |
| 治理 | `contracts/governance/` | `packages/governance/` | — | `packages/governance/` |
| 代理 / 升级 | `contracts/proxy/` | `packages/upgrades/` | `contracts/src/proxy/` | `packages/contract-utils/` |
| 工具 / 安全 | `contracts/utils/` | `packages/utils/`、`packages/security/` | `contracts/src/utils/` | `packages/contract-utils/` |
| 账户 | `contracts/account/` | `packages/account/` | — | `packages/accounts/` |

搜索某个组件时，优先浏览这些路径。

**Sui Move** 不在上面的固定目录表格中，并且没有 `@openzeppelin/contracts-cli` 生成器，因此应始终使用前面介绍的模式发现方法 —— 将某个 package 的 `examples/` 作为规范集成示例进行适配，并通过 MVR import，而不是复制源码。其余信息（package 集合、组合方式、代码风格约定、精确 API 和工具链）都应从库自身的元数据中发现，首先查看 [`llms.txt`](https://raw.githubusercontent.com/OpenZeppelin/contracts-sui/main/llms.txt)；`setup-sui-contracts` skill 覆盖完整的项目设置、依赖、`--build-env` 构建和质量门禁流程。

### 已知的版本特定注意事项

不要根据过去的知识假设 override 点 —— 必须始终通过阅读项目当前安装的源码进行确认。旧版本中标记为 `virtual` 的函数，在新版本中可能已经不再是 `virtual`，因此不能继续 override。源码中的 NatSpec 会指出正确的 override 点（例如：`NOTE: This function is not virtual, {X} should be overridden instead`）。

一个已知示例是：Solidity ERC-20 的 transfer hook 在 v4 和 v5 之间发生了变化。在建议 override 之前，必须阅读项目当前安装的 `ERC20.sol`，确认哪个函数实际被标记为 `virtual`。

## CLI 生成器

`@openzeppelin/contracts-cli` 包可以通过命令行生成 OpenZeppelin 合约参考实现。只要目标合约类型存在对应命令，就应在“生成 → 比较 → 应用”工作流中把它作为参考来源。

### 发现命令和选项

运行 `npx @openzeppelin/contracts-cli --help` 列出所有可用命令。每个命令对应一种合约类型（例如 `solidity-erc20`、`cairo-erc721`、`stellar-fungible`）。运行 `npx @openzeppelin/contracts-cli <command> --help` 查看该命令可用的选项。不要依赖过去的知识来判断有哪些选项；由于 CLI 可能已经更新，因此每次会话开始时都应先检查 `--help`。

### “生成 → 比较 → 应用”快捷流程

当目标合约类型存在 CLI 命令时，将生成结果输出到临时文件并进行 diff，从而避免把大量生成的合约代码放进对话上下文：

1. **生成基础版本** —— 只使用必需参数，关闭所有可选功能，并输出到文件：
   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK > /tmp/oz-baseline.sol
   ```
2. **生成启用目标功能的版本** —— 再运行一次，并启用需要的功能，输出到第二个文件：
   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK --pausable > /tmp/oz-variant.sol
   ```
3. **比较** —— 对两个文件执行 diff，准确识别发生了哪些变化（import、继承、state、constructor、函数、modifier）：
   ```bash
   diff /tmp/oz-baseline.sol /tmp/oz-variant.sol
   ```
4. **应用** —— 修改用户现有合约，加入发现的这些变化

对于会相互影响的功能（例如访问控制 + 可升级性），还应额外生成一个同时启用这些功能的组合版本进行比较。

### 当不存在 CLI 命令，或目标功能未被 CLI 覆盖时

没有 CLI 命令**并不代表**库不支持该功能，只代表该合约类型没有生成器。此时必须退回使用 [模式发现与集成](#模式发现与集成) 中的通用方法。

同样，如果某个合约类型存在 CLI 命令，但 CLI 没有暴露某项具体功能对应的选项，也不要停在这里。针对该功能退回模式发现流程：阅读项目中已安装的库源码，找到相关组件，提取它的集成要求，然后应用到用户合约中。
