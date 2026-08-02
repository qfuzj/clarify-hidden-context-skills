# Clarify Hidden Context

> 先追问、后实施——逐题澄清那些"你已知道但还没告诉 AI"的关键上下文，并将结论写回项目文件。

`clarify-hidden-context` 是一个面向 Codex / Claude Code 的需求澄清 skill。它在 AI 开始设计方案或编写代码前，先识别会显著改变实现路径的未说明信息，并通过逐题追问的方式补齐决策上下文。与常见的"一次性输出所有问题"不同，它采用**对话式逐一追问**，避免信息过载；澄清结论**实时写回 `CONTEXT.md` 或 `docs/decisions/`**，跨会话可复用。

## 它解决什么问题

AI 辅助开发中，多数失败不是代码不会写，而是上下文没有对齐。开发者已知的目标用户、使用路径、技术约束、完成标准、安全要求等，如果没有显式传达给 AI，就会带来返工和安全风险。

这个 skill 的核心目标：**把它们提前显性化，并且只有会改变决策的问题才问**。

## 核心设计（v2.0 重构）

v2.0 对比 [mattpocock/skills](https://github.com/mattpocock/skills) 中的 `grill-with-docs` 进行了全面 benchmark，吸收其**组合式架构、逐题追问、持久化产出**三项优势，同时保留并强化了**三级优先级、安全边界、防偏规则**这些对方缺失的能力。

### 五项核心原则

1. **只追问会改变决策的问题** — 不问能从代码/文档查到的，不问答案不影响实现路径的
2. **一次只问一个** — 等待回答后再继续，不给用户堆砌问题清单
3. **按优先级区分** — [必须确认] 涉及安全/付费/不可逆/权限；[建议确认] 有安全默认值；其余列出假设
4. **写回项目文件** — 术语定义写入 `CONTEXT.md`，架构决策写入 `docs/decisions/`，产出可跨会话复用
5. **与代码交叉验证** — 用户声称的规则与代码矛盾时立即指出

### 安全边界

以下情况**必须明确确认，不得使用默认假设**：

- 生产环境数据、用户隐私、敏感信息
- 调用付费 API、云服务、第三方接口
- 不可逆操作（删除、覆盖、发布）
- 修改权限、认证、授权逻辑
- 影响安全、合规、审计要求的决策

## 两阶段工作流

```
┌─────────────────────────────────────────────────────┐
│  第一阶段：逐题追问                                    │
│                                                     │
│  自查代码 → 筛出高影响未知项 → 逐题提问 → 写回结论       │
│                                                     │
│  ⚠️ 强制停止等待用户回复，不跳过不偷跑                   │
└────────────────────────┬────────────────────────────┘
                         │ 用户补充信息或回复"继续"
                         ▼
┌─────────────────────────────────────────────────────┐
│  第二阶段：形成方案或实施                               │
│                                                     │
│  合并事实+答案+假设 → 输出方案/实施代码 → 标注残余风险     │
└─────────────────────────────────────────────────────┘
```

第一阶段采用对话式逐一提问（借鉴 grilling 的 "one at a time, waiting for feedback"），不再是过去的三段式固定输出。问题多时仍可使用分段格式汇总，但优先对话式。

第二阶段的输出取决于用户任务类型：评审/诊断 → 只输出分析；构建/修改 → 实施并验证。

## 适用场景

| 场景 | 示例 |
|------|------|
| 新功能/MVP 开发 | "开发 AI 知识库，先问关键问题" |
| 现有仓库增加模块 | "给 CLI 加 CSV 导出，先查代码再提问" |
| 系统设计评审 | "评审这个架构，补齐上下文给建议" |
| AI Agent/工具构建 | "构建自动化 pipeline，确认边界条件" |
| Vibe coding | "快速补齐需求上下文再动手" |

## 安装

### Codex

**方式一：Skill Installer（推荐）**

```text
使用 $skill-installer 安装以下 GitHub 仓库中的 skill：

仓库：https://github.com/qfuzj/clarify-hidden-context-skills
skill 路径：.
安装名称：clarify-hidden-context
```

**方式二：手动安装（macOS / Linux）**

```bash
# 用户级（本机所有项目可用）
mkdir -p "$HOME/.agents/skills"
git clone https://github.com/qfuzj/clarify-hidden-context-skills.git \
  "$HOME/.agents/skills/clarify-hidden-context"

# 仓库级（仅当前项目）
mkdir -p .agents/skills
git clone https://github.com/qfuzj/clarify-hidden-context-skills.git \
  .agents/skills/clarify-hidden-context
```

**方式三：手动安装（Windows PowerShell）**

```powershell
# 用户级
$dest = Join-Path $HOME ".agents\skills\clarify-hidden-context"
New-Item -ItemType Directory -Force -Path (Split-Path $dest) | Out-Null
git clone https://github.com/qfuzj/clarify-hidden-context-skills.git $dest

# 仓库级
$dest = Join-Path (Get-Location) ".agents\skills\clarify-hidden-context"
New-Item -ItemType Directory -Force -Path (Split-Path $dest) | Out-Null
git clone https://github.com/qfuzj/clarify-hidden-context-skills.git $dest
```

### Claude Code

```bash
# 用户级（本机所有项目可用）
mkdir -p ~/.claude/skills
git clone https://github.com/qfuzj/clarify-hidden-context-skills.git \
  ~/.claude/skills/clarify-hidden-context
```

## 快速开始

```text
使用 $clarify-hidden-context 帮我处理下面的开发任务。

任务：开发面向小团队的 AI 会议纪要 Web 应用。
已知：Next.js，调用大模型 API，先做 MVP。

先做隐藏区检查，不要输出代码。等我补充信息后再给方案。
```

skill 会先自查代码（确认 Next.js 项目结构），然后逐题提问高影响未知信息，每轮等待你的回答。澄清结论（如术语定义、架构决策）实时写回项目文件。

## 控制语句

| 目标 | 推荐表达 |
|------|---------|
| 第一阶段后暂停 | `先只做隐藏区检查，等我回复后再继续。` |
| 限制问题数量 | `只问最关键的 3 个问题。` |
| 自动采用默认值 | `没有阻塞项时按默认假设继续。` |
| 避免重复提问 | `先检查仓库，不要问能从代码中确认的信息。` |
| 只输出设计 | `只给方案，不要修改文件。` |

## 项目结构

```
clarify-hidden-context/
├── SKILL.md           # 入口：核心原则、两阶段工作流、写回规则（52 行）
├── dimensions.md      # 参考：7 个决策维度及筛选标准
├── format-guide.md    # 参考：输出格式示例与反模式
├── safety-rules.md    # 参考：安全边界详解与默认假设指南
├── README.md          # 项目文档
└── agents/
    └── openai.yaml    # Codex 集成配置
```

### 为什么拆成多个文件

按 Anthropic Claude Code 官方最佳实践：SKILL.md 保持简洁（≤500 行，越短越好），详细参考材料放入支持文件按需加载。这样每次调用只消耗核心指令的 token，参考内容只在需要时由模型主动读取。

## 与 grill-with-docs 的对比

| 维度 | clarify-hidden-context | grill-with-docs (mattpocock) |
|------|:---:|:---:|
| 提问方式 | 对话式逐题追问 | 逐题追问（一致 ✅） |
| 持久化产出 | CONTEXT.md + 决策记录 | CONTEXT.md + ADR（一致 ✅） |
| 安全边界 | 三级优先级 + 强制确认清单 | ❌ 无 |
| 模块化 | 入口 + 3 参考文件 | 2 技能组合（grilling + domain-modeling） |
| Token 效率 | 52 行入口 | 86 行组合入口 |
| 防偏规则 | ✅ 6 条 | ⚠️ 部分（ADR 创建条件） |

## 常见问题

### 会不会每次都阻塞开发？

不会。只有使方案失效或带来明显风险的信息才属"必须确认"。低风险项列出默认假设后直接推进。

### 支持哪些 AI 工具？

当前主要适配 Codex 和 Claude Code。安装到对应 skills 目录即可使用。SKILL.md 遵循 [Agent Skills](https://agentskills.io) 开放标准，理论上兼容所有实现该标准的工具。

### 隐藏区和心理分析的关系？

这里只用"隐藏区"做需求澄清的类比。skill 不会分析用户心理或评价认知能力，始终使用"可能""尚未明确"等措辞。

### 能代替产品/安全评审吗？

不能。它用于尽早发现缺失上下文和风险，但不能替代正式的产品决策、安全审计或合规评估。

## 贡献

欢迎通过 Issue 或 PR 提交改进。修改 skill 时请保持 SKILL.md 简洁（目标 60 行以内），详细内容放入参考文件。
