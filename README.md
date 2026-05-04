# solidity-skills

一个用于存放 Solidity 相关 Codex 技能的仓库。

foundry-test -> solidity-foundry-code-review -> foundry-post-dev-optimization -> solidity-foundry-security-review -> doc-natspec

当前仓库内置五个技能：

- `doc-natspec`：用于在 Solidity / Foundry 开发完成后补齐和修复 NatSpec 注释，并用 `forge doc` 做最终验证
- `foundry-test`：用于在 Foundry 项目开发完成后补齐和强化测试，优先补高价值单元测试，并在必要时增加集成、fuzz 与 invariant 测试
- `solidity-foundry-code-review`：用于在 Foundry 项目功能开发后或合并前做工程质量 code review，关注正确性、边界条件、接口设计、集成假设、可维护性和测试缺口
- `foundry-post-dev-optimization`：用于在 Foundry 项目开发完成后做 gas、代码结构和可维护性优化，低风险项直接修，风险较高的项保留为建议
- `solidity-foundry-security-review`：用于在 Foundry 项目中进行面向代码上下文的安全审查，强调先读项目、再做结论

## 当前内容

```text
skills/
  doc-natspec/
    SKILL.md
  foundry-test/
    SKILL.md
  solidity-foundry-code-review/
    SKILL.md
  foundry-post-dev-optimization/
    SKILL.md
  solidity-foundry-security-review/
    SKILL.md
```

## 技能定位

`doc-natspec` 适用于以下场景：

- Solidity / Foundry 开发完成后补齐 NatSpec
- 检查 `public` / `external` 函数注释是否完整
- 为复杂的 `internal` / `private` 函数补写必要注释
- 为需要被外部理解的 `struct` 和 `event` 编写说明
- 在 `forge doc` 前做文档生成兼容性检查

`foundry-test` 适用于以下场景：

- Solidity / Foundry 开发完成后补齐或强化测试
- 扫描 `src/` 与现有 `test/` 识别高价值测试缺口
- 优先补充单元测试，并在必要时补集成、fuzz 与 invariant 测试
- 用 `forge test` 验证新增测试是否成立

`foundry-post-dev-optimization` 适用于以下场景：

- Solidity / Foundry 功能开发完成后做专项优化收尾
- 默认扫描 `src/` 下所有合约，排除接口，识别 gas、结构和可维护性优化点
- 低风险优化直接落地，中高风险优化按条件执行或保留为建议
- 优先复用项目已有 benchmark、snapshot 或 gas report 流程验证优化收益
- 在不改变预期行为的前提下收敛代码质量和执行成本

`solidity-foundry-code-review` 适用于以下场景：

- 在功能开发后或合并前做 Solidity / Foundry 工程质量审查
- 检查正确性、边界条件、状态流、接口设计和依赖集成是否符合预期
- 发现测试缺口、部署假设、初始化顺序和调用路径风险
- 输出基于文件和行号的 review findings，并把安全、测试或优化专项问题路由到对应 skill

`solidity-foundry-security-review` 适用于以下场景：

- 对 Foundry Solidity 项目做安全检查
- 在功能开发后做合约 hardening
- 在合并前进行代码审查
- 分析 vault、token 集成、升级代理、oracle 依赖和低级调用等风险面

## 设计原则

- 先读项目：优先阅读 `README.md`、`foundry.toml`、目标合约和相关测试
- 基于代码：所有结论都应落到具体文件和实现
- 面向利用路径：关注攻击路径、失败路径与资金影响
- 可验证：修复建议应尽量附带 Forge 测试方向

## 使用方式

```
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/doc-natspec
```

```
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/foundry-test
```

```
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/solidity-foundry-code-review
```

```
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/foundry-post-dev-optimization
```

```
$skill-installer install https://github.com/dericklau3/solidity-skills/tree/main/skills/solidity-foundry-security-review
```
