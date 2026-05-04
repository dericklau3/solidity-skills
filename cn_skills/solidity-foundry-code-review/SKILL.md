---
name: solidity-foundry-code-review
description: "在 Foundry 项目中审查 Solidity 合约工程质量。适用于用户希望在合并前或功能开发后获得基于文件的 code review，覆盖正确性、边界条件、可维护性、接口设计、调用流程、部署假设、依赖集成或测试缺口，但未要求完整安全审计或优化 pass。"
license: AGPL-3.0-only
metadata:
  author: derick
---

# 审查 Solidity 和 Foundry 代码质量

## 核心工作流

### 审查前先理解请求

当用户要求 code review、merge review、工程审查、PR review，或对 Solidity / Foundry 代码做开发后的质量检查时，使用这个 skill。

如果请求明确是安全审计，使用安全审查 skill。如果请求是添加测试，使用 Foundry 测试 skill。如果请求是开发后的 Gas 或可维护性优化，使用优化 skill。Code review 可以识别这些方向，但除非用户要求，不要扩展成完整审计、测试编写 pass 或优化 pass。

### 总是先阅读项目

在报告发现前：

1. 如果存在，阅读 `README.md`、`foundry.toml` 和项目文档。
2. 搜索 `src/`、`test/`、`script/`、interfaces、libraries、mocks、fixtures 和部署 helper。
3. 阅读范围内文件，以及理解跨合约行为所需的相邻文件。
4. 阅读测试以推断预期行为，但不要假设测试完整或正确。
5. 当 import、继承或依赖行为重要时，阅读 `remappings.txt`、`lib/`、`node_modules/` 和包配置。

如果某个文件无法读取，明确指出路径，并说明这如何限制审查置信度。不要退回成通用 checklist，却假装已经审查过项目。

### 审查姿态

默认采用 code review 姿态：

* 先给 actionable findings，并按严重程度排序。
* 优先关注正确性、行为回归、不清晰假设、缺失校验、状态流错误、集成错误和测试缺口。
* 每个发现都要基于具体文件和调用路径。
* 除非风格问题会造成真实维护风险，否则不要报告纯风格 nit。
* 不要把推测性问题当作强发现。降低置信度，或列为剩余风险。

## 审查方法

### Step 1: 重建意图和范围

建立一个紧凑模型：

* 面向用户和特权角色的 entrypoint
* 预期状态转换和调用顺序
* 资产、余额、角色、配置和部署假设
* 外部集成和依赖提供的行为
* 定义或暗示预期行为的测试

如果审查 diff，将改动行为和现有系统对比。如果审查整个项目，识别最重要的合约和流程。

### Step 2: 检查正确性和边界情况

寻找项目相关的问题，例如：

* 无效或缺失的前置条件
* 状态更新顺序错误
* 会计不一致或缓存值过期
* 围绕零值、最大值、空数组、重复输入、取整、decimals、时间或区块假设的边界错误
* 等价路径之间权限检查不一致
* event 缺失、误导，或触发时机错误
* revert 行为让集成变脆弱或含义不清
* 部署或初始化路径可能让系统处于部分配置状态

这不是漏洞 checklist。每个问题都要和项目预期行为关联。

### Step 3: 检查接口和集成

审查合约如何暴露和消费行为：

* public 和 external API：命名、返回值、revert surface、caller 预期和向后兼容性
* 内部边界：helper 职责、重复逻辑和不清晰抽象
* 依赖集成：必需 override、hook、modifier、constructor、initializer、storage 和版本特定行为
* script 和部署 helper：配置一致性、初始化顺序、环境假设和可重复性
* 测试和 mock：是否覆盖真实用户和操作方路径

在声明 OpenZeppelin、Solady、Chainlink、Uniswap、代理库、Token 标准或其他导入行为之前，先阅读项目安装的依赖源码。

### Step 4: 区分审查类型

按正确的后续动作分类：

* **Code review finding** - 合并前应修复的正确性、接口、可维护性或集成问题。
* **Security follow-up** - 可能的攻击路径或信任边界问题，需要安全审查 skill 或更深的对抗分析。
* **Test follow-up** - 缺失或薄弱的测试覆盖，应交给 Foundry 测试 skill。
* **Optimization follow-up** - 更适合由优化 skill 处理的 Gas、结构或可维护性改进。

不要混淆这些类别。Code review 可以路由工作，但不要假装已经完成所有专门审查。

### Step 5: 推荐最小修复和测试

对于每个可执行发现：

* 描述最小的行为保持修复。
* 优先使用项目已有模式和已安装依赖，而不是新建自定义抽象。
* 标注任何 storage layout、initializer、ABI 或集成兼容性风险。
* 当问题属于行为问题时，建议聚焦的 Forge 回归测试。
* 如果修复方案存在歧义，指出需要的设计决策，不要编造一个。

## 验证

当 code review 包含代码改动时，先运行最窄相关的 `forge test`，可行时再运行完整测试套件。

对于只读 review，除非实际运行过测试，否则不要声称测试通过。可以说明哪些验证命令会提升置信度。

如果 RPC 访问、依赖项、环境变量或 fork 配置阻塞验证，报告具体命令和阻塞原因。

## 输出约定

默认审查输出：

### Findings

按严重程度排序：`Critical`、`High`、`Medium`、`Low`，必要时再给非阻塞 notes。

每个发现包含：

* 严重程度
* 受影响文件和行号，或紧凑代码位置
* 问题
* 影响
* 推荐修复
* 有用时建议 Forge 测试
* 有用时给出置信度

### Open Questions or Assumptions

只列出会实质影响正确性或合并就绪度的问题。

### Summary

简要说明已审查文件、执行的验证和剩余风险。

如果没有发现实质性问题，明确说明，并列出测试缺口或未审查表面。

## 回复风格

* 发现优先
* 简洁，并基于具体文件
* 不输出通用 checklist
* 避免过度表扬式 review summary
* 区分已确认发现和建议
* 不要要求用户检查你自己可以检查的文件
