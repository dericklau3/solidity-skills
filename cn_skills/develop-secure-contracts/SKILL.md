---
name: develop-secure-contracts
description: "用于在实际使用 Solidity 0.8.30 或更新编译器的项目中集成或扩展 OpenZeppelin Contracts。"
license: AGPL-3.0-only
metadata:
  author: OpenZeppelin
---

# 集成 OpenZeppelin Contracts

## 遵守本技能

执行时遵守本技能适用的范围、执行规则、验证要求和输出约定。不静默跳过必需步骤，不用建议代替已授权的实现。用户明确指令及更高优先级指令优先。必需步骤无法完成时，说明具体限制、继续独立工作，不宣称该步骤已经完成。

## 范围

本技能适用于实际使用 **Solidity 0.8.30 或更新版本**编译的项目。读取 pragma 与实际编译器配置，宽泛 pragma 本身不足以证明版本。如果配置版本更旧，说明不匹配并采用适合该版本的指导，不套用本技能的版本专属规则，也不静默升级编译器。

概念问题直接解释。实现请求在已授权范围内直接修改现有合约；只有用户要求替换时才替换。用户明确指令优先于技能默认规则，沿用已有授权，从项目上下文解决常规选择。

## 确定集成方式

1. 使用可用文件搜索工具定位范围内的 `.sol`，例如 `rg --files -g '*.sol'`。阅读目标合约、相关测试和编译器/包配置。通过 remapping、import 定位实际安装依赖，使用 `contracts-upgradeable` 时也要解析对应依赖。
2. 阅读相关组件、NatSpec 及有帮助的本地示例/测试。根据该版本确认 API、`virtual` 扩展点、hook、modifier、继承、constructor/initializer 参数和 storage 要求，不根据 v4 习惯推断 v5 hook。
3. 优先采用能满足请求的现有组件，通过 import 组合/继承，或仅使用支持的扩展点。只有已安装库不能提供所需行为时才写自定义逻辑。不复制库源码，也不重复实现已有权限、暂停或接口检测机制。
4. 确定最小兼容改动：import、继承、override、guard 和初始化。检查 override 冲突、安全组件重复，以及对 caller、event、error、脚本、测试和文档的影响。

缺少依赖时，检查声明版本/lockfile，仅将匹配版本的官方源码作为明确披露的备用依据。无法确定安装版本意味着证据不足，不代表可以假设最新 API。文件无法读取时说明路径和原因，继续独立工作，仅在缺失信息影响正确性时询问。

## 实施改动

保留无关修改、请求之外的业务语义和项目约定。在现有代码中完成集成，不生成替代脚手架或顺手清理相邻代码风格。

可升级合约需检查部署上下文、storage layout、继承顺序、namespaced storage/gap、initializer/reinitializer 顺序和升级授权。已有代理的新实现必须兼容该代理的状态。宣称兼容前，对照已部署基线验证或使用项目升级验证器；测试通过不足以证明。缺少基线证据时明确标记未验证。

### Solidity 0.8.30+ 校验风格

新增或实质修改的简单 guard 优先使用 `require(validCondition, Errors.Xxx(...))`。复用项目的 `Errors` 命名空间；新增错误优先放在共享 `Errors.sol`，已有其他约定时遵循项目。

```solidity
require(account != address(0), Errors.ZeroAddress());
require(balance >= amount, Errors.InsufficientBalance(balance, amount));
```

保留现有 custom error，不改成 revert string。依赖分支的失败逻辑可保留 `if (...) revert ...`；用户明确指定风格时遵循用户。不仅为统一语法而触碰无关 guard。

`require` 会无条件求值所有参数。转换 `if/revert` 前检查错误参数求值，包括可能 revert、有副作用或有明显成本的调用。转换会改变行为或引入不合理工作时，保留条件求值。

### 可选 CLI 参考

仅当生成示例有助于解决集成疑问时使用 `@openzeppelin/contracts-cli`。通过 `--help` 确认命令和参数，项目已固定版本时使用该版本。按需将基础版和功能版生成到临时文件，对比后仅应用经实际安装源码验证的相关变化。CLI 输出不能覆盖项目依赖版本这一事实依据。CLI 没有对应命令不代表库缺少组件，继续通过源码集成。

## 验证与完成

通过项目正常构建/测试命令完成编译，运行变更行为的聚焦集成/回归测试。同步受影响的 caller、脚本、测试和文档，检查修改文件格式和 diff。

共享继承、初始化、权限、会计或公共行为变更要覆盖全部受影响套件；无法限定影响面时运行全量。完成明确的全量请求及项目必需检查。通过后，仅因新增改动、失败或未解决疑点重复或扩大验证。将基线失败、环境阻塞与回归分开报告。

最终说明已实现行为、与改动相关的安装版本证据、实际验证及剩余兼容性或执行限制。回复长度与改动规模相称。
