# solidity-skills

一个用于存放 Solidity 相关 Codex 技能的仓库。

当前仓库内置的技能是 `solidity-foundry-security-review`，用于在 Foundry 项目中进行面向代码上下文的安全审查。它强调先读项目、再做结论，输出基于文件、函数、调用路径和测试建议的审计结果，而不是泛化的清单式建议。

## 当前内容

```text
skills/
  solidity-foundry-security-review/
    SKILL.md
```

## 技能定位

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

将仓库中的技能目录接入你的 Codex 技能体系后，即可在需要进行 Solidity / Foundry 安全审查时调用该技能。

如果后续要扩展更多技能，建议继续按 `skills/<skill-name>/SKILL.md` 的目录结构维护。

## License

本项目采用 `AGPL-3.0-only`，详见 [LICENSE](/Users/derick/Documents/git-project/contracts/evm/solidity-skills/LICENSE)。
