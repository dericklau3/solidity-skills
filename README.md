# solidity-skills

一个用于存放 Solidity / Foundry 相关 Codex 技能的仓库。

推荐的开发后检查流程：

```text
develop-secure-contracts
        ↓
foundry-test
        ↓
foundry-post-dev-optimization
        ↓
solidity-foundry-security-review
        ↓
根据审查结果修复并补回归测试
        ↓
doc-natspec
```

当前仓库内置五个技能：

- `develop-secure-contracts`：用于 Solidity 0.8.30+ 项目中集成和扩展 OpenZeppelin Contracts，强调先读取项目实际依赖与编译器配置，再基于已安装库做最小化集成
- `foundry-test`：用于在 Foundry 项目开发完成后补齐和强化高价值测试，包括单元、集成、fuzz、invariant 与回归测试，并要求把测试目的直接写在 `test...()` 函数顶部
- `foundry-post-dev-optimization`：用于在 Foundry 项目开发完成后做 gas、语义 no-op 裁剪、代码结构和可维护性优化，低风险项直接修，风险较高的项按证据与验证强度决定是否执行
- `solidity-foundry-security-review`：用于 Foundry Solidity 项目的定向安全审查或完整协议审计，强调协议建模、资产流、会计、经济攻击、权限、依赖与可验证攻击路径
- `doc-natspec`：用于开发完成后补齐和修复 NatSpec，支持项目自有接口、继承文档与 `@inheritdoc`，并用 `forge doc` 做最终验证

## 当前内容

```text
skills/
  develop-secure-contracts/
    SKILL.md
  doc-natspec/
    SKILL.md
  foundry-test/
    SKILL.md
  foundry-post-dev-optimization/
    SKILL.md
  solidity-foundry-security-review/
    SKILL.md
    references/
      protocol-audit-methodology.md
```

## 技能定位

### `develop-secure-contracts`

适用于：

- Solidity `0.8.30+` 项目
- ERC20、ERC721、ERC1155 等 OpenZeppelin token 组件集成
- `Ownable`、`AccessControl`、`AccessManager` 等权限控制
- `Pausable`、`ReentrancyGuard` 等安全组件
- Governor、Timelock、Accounts 等 OpenZeppelin 组件
- 在已有项目中基于实际安装的 OpenZeppelin 版本做集成，而不是凭记忆假设 API

主要原则：

- 先确认实际 Solidity 编译器版本与配置
- 先读项目已有代码和已安装 OpenZeppelin 源码
- 优先使用库组件，不重复手写已有能力
- CLI 生成结果只作为参考，项目实际安装的依赖源码才是最终依据
- 对 Solidity 0.8.30+ 的简单校验，优先使用 `require(condition, CustomError())` 风格，同时尊重项目已有约定

### `foundry-test`

适用于：

- Solidity / Foundry 开发完成后补齐或强化测试
- 扫描 `src/` 与现有 `test/` 识别高价值测试缺口
- 优先补充单元测试，并在必要时增加集成、fuzz、invariant 与真实流程测试
- 为安全问题或 bug 添加回归测试
- 用 `forge test` 验证新增测试是否成立

测试书写约定：

- 每个新增或实质修改的 `test...()` 函数，在函数体最前面用简短注释写明该测试要证明的行为、保证或失败条件
- fuzz 测试说明要验证的 property 和主要输入范围
- invariant 测试说明在任意调用序列下必须长期成立的协议属性
- 不生成独立的 `test/docs` 测试说明文件

### `foundry-post-dev-optimization`

适用于：

- Solidity / Foundry 功能开发完成后做专项优化收尾
- 默认扫描 `src/` 下的合约，识别 gas、语义 no-op、结构和可维护性优化点
- 从 caller、数据流、控制流、类型约束、状态不变量、接口契约和副作用中证明非平凡删除不会改变可观察行为
- 优先复用项目已有 benchmark、snapshot 或 gas report 流程验证优化收益

主要原则：

- 低风险、可证明的优化可以直接落地
- `assembly` 等高风险优化保持保守，默认作为建议项
- 不把需要 memory array 手动截断等通常依赖 assembly 的技巧作为默认推荐模式
- 没有 before/after 实测时，不把优化描述成“已测得的 gas 改善”，而应写成预期优化、可能优化或结构简化

### `solidity-foundry-security-review`

适用于：

- 对 Foundry Solidity 项目做安全检查
- 在功能开发后做合约 hardening
- 在合并前进行定向安全审查
- 对完整协议进行 repository-wide / protocol-wide audit
- 分析 vault、token 集成、升级代理、oracle、外部协议、回调、签名、低级调用等风险面

主要原则：

- 不从漏洞清单开始，而是先理解协议目标、生命周期、资产流、角色、会计与关键不变量
- 区分定向审查和完整协议审计，不把局部检查包装成全量覆盖
- 对重大 finding 要给出可达路径、实际影响，以及尽可能具体的 Forge PoC、数值例子或状态证明
- 测试只作为证据之一，不把“测试通过”当成“没有漏洞”的证明

### `doc-natspec`

适用于：

- Solidity / Foundry 开发完成后补齐或修复 NatSpec
- 检查 `public` / `external` API 是否有准确文档
- 为复杂的 `internal` / `private` 函数补写必要注释
- 为需要被外部理解的 `struct` 和 `event` 编写说明
- 检查项目自有 interface / base contract 的继承文档
- 在适合的 override 上使用继承 NatSpec 或 `@inheritdoc`，避免机械复制 `@notice` / `@param` / `@return`
- 用 `forge doc` 做最终验证

主要原则：

- 默认检查项目自有接口，但不机械修改 vendored dependency
- override 与父级语义一致时优先复用继承文档
- override 改变权限、行为、side effect、revert、会计或集成语义时必须补充本地文档
- 不为了凑 tag 数量而复制低信息注释

## 设计原则

- **先读项目**：优先阅读 `README.md`、`foundry.toml`、目标合约、测试、脚本和实际安装的依赖
- **基于代码**：所有结论尽量落到具体实现，不用记忆中的库 API 代替项目事实
- **职责分离**：测试、优化、安全审查和 NatSpec 各自保持明确边界
- **行为优先**：测试与优化围绕可观察行为、状态与协议保证，而不是机械追求数量
- **面向利用路径**：安全审查关注攻击路径、失败路径、资金影响与破坏的不变量
- **可验证**：修改后使用项目已有的 Forge 流程、测试、benchmark 或文档构建方式验证

## 使用方式

```bash
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/develop-secure-contracts
```

```bash
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/foundry-test
```

```bash
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/foundry-post-dev-optimization
```

```bash
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/solidity-foundry-security-review
```

```bash
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/doc-natspec
```
