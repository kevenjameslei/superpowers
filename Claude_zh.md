# Superpowers 项目技术分析（中文）

> 分析日期：2026-03-30
> 项目版本：v5.0.6
> 分析模型：Claude Sonnet 4.6

---

## 一、项目概述

**项目名称**: Superpowers
**当前版本**: 5.0.6 (2026-03-24)
**开发者**: Jesse Vincent (Prime Radiant)
**许可证**: MIT
**仓库**: https://github.com/obra/superpowers

### 项目定位

Superpowers 是一个为 AI 编码代理构建的完整软件开发工作流框架。它通过可组合的"技能"(Skills)系统和初始指令，使编码代理能够系统性地完成从需求梳理到代码实现、测试、审查、部署的全生命周期软件工程工作。

**核心理念**：从创意到设计，再到实现的完整协作工作流，强调测试驱动开发(TDD)、系统化调试、证据优先于断言。

---

## 二、项目整体结构

```
superpowers/
├── package.json                    # Node.js 项目配置
├── README.md                       # 主文档
├── CHANGELOG.md                    # 版本变更日志
├── RELEASE-NOTES.md               # 发布说明
├── LICENSE                        # MIT 许可证
├── .claude-plugin/                # Claude Code 插件配置
│   └── plugin.json
├── .cursor-plugin/                # Cursor 编辑器插件配置
│   └── plugin.json
├── .codex/                        # Google Codex 支持
│   └── INSTALL.md
├── .opencode/                     # OpenCode 平台支持
│   ├── INSTALL.md
│   └── plugins/
│       └── superpowers.js         # OpenCode 插件实现
├── skills/                        # 14 个核心技能库
│   ├── brainstorming/
│   ├── writing-plans/
│   ├── subagent-driven-development/
│   ├── executing-plans/
│   ├── test-driven-development/
│   ├── using-git-worktrees/
│   ├── systematic-debugging/
│   ├── requesting-code-review/
│   ├── receiving-code-review/
│   ├── dispatching-parallel-agents/
│   ├── verification-before-completion/
│   ├── finishing-a-development-branch/
│   ├── writing-skills/
│   └── using-superpowers/
├── agents/                        # 子代理角色定义
│   └── code-reviewer.md
├── commands/                      # 快捷命令
│   ├── brainstorm.md
│   ├── write-plan.md
│   └── execute-plan.md
├── hooks/                         # 生命周期钩子
│   ├── hooks.json                 # Claude Code 钩子配置
│   ├── hooks-cursor.json          # Cursor 钩子配置
│   ├── session-start              # 会话启动脚本
│   └── run-hook.cmd               # Windows 钩子运行器
├── docs/                          # 文档和规范
│   ├── README.codex.md
│   ├── README.opencode.md
│   └── superpowers/
│       ├── plans/                 # 实现计划示例
│       └── specs/                 # 设计规范示例
├── tests/                         # 测试用例
│   ├── brainstorm-server/
│   ├── subagent-driven-dev/
│   ├── explicit-skill-requests/
│   └── skill-triggering/
└── .github/                       # GitHub 工作流
    ├── ISSUE_TEMPLATE/
    └── PULL_REQUEST_TEMPLATE.md
```

**项目规模**：
- 14 个核心技能，共计约 3157 行文档
- 支持 4 个平台：Claude Code、Cursor、Google Codex、OpenCode
- 主要由 YAML 前缀 + Markdown 内容组成的技能文档
- 轻量级 JavaScript/Node.js 实现（OpenCode 插件）

---

## 三、核心技能体系（Skills Library）

### 3.1 技能分类与层级

14 个核心技能按工作流顺序排列如下：

#### 阶段 1：需求梳理

**1. brainstorming（头脑风暴）**
- 触发条件：创建特性、构建组件、修改行为
- 核心过程：Socratic 对话 → 需求精化 → 2-3 种方案对比 → 设计文档编写 → 用户审核
- 输出：设计规范文档（`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`）
- **关键门控**：必须生成并获得用户批准的设计，即使是"简单"项目也不例外

#### 阶段 2：隔离与计划

**2. using-git-worktrees（Git 隔离工作区）**
- 触发：设计批准后，实施前
- 功能：创建隔离 git worktree、自动运行项目设置、验证基线测试
- 优先级：`.worktrees/` > `worktrees/` > `~/.config/superpowers/worktrees/`
- 安全检查：确保项目本地目录被 `.gitignore` 忽略

**3. writing-plans（编写实现计划）**
- 前置：已批准设计、已建立 worktree
- 核心原则：假设工程师零上下文，编写咬度适当的任务（2-5 分钟一个）
- 任务结构：完整的代码块、精确的文件路径、确切的命令、预期输出
- **无占位符规则**：禁止 TBD、TODO、"implement later"、"类似于任务 N"
- 输出：实现计划（`docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`）
- 验证：计划自审查（规范覆盖、占位符扫描、类型一致性）

#### 阶段 3：实施与迭代

**4. subagent-driven-development（子代理驱动开发）** 【推荐方式】
- 执行模式：为每个任务派遣专用子代理，两阶段审查（规范符合性 + 代码质量）
- 流程：实施 → 自审查 → 规范审查 → （若有问题）修复 → 代码质量审查 → 完成
- 优势：新会话上下文、自动检查点、快速迭代、避免上下文污染
- 模型策略：简单任务用便宜模型，复杂任务用强大模型

**5. executing-plans（执行计划）** 【备选方式】
- 执行模式：在独立会话中批量执行，有审查检查点
- 适用：不支持子代理的平台
- 流程：加载 → 批判性审查 → 逐任务执行 → 验证 → 完成

**6. test-driven-development（测试驱动开发）**
- 触发：实施任何特性、修复任何 bug
- 循环：RED（失败测试）→ GREEN（最小代码）→ REFACTOR（清理）
- **铁律**：没看到测试失败就不知道它测试的是什么
- 代码删除：测试前写的代码必须删除并重新实现

#### 阶段 4：质量保障

**7. systematic-debugging（系统化调试）**
- 触发：任何 bug、测试失败、不可预期行为
- 四阶段：① 根因调查 → ② 防御-深度策略 → ③ 条件基础等待 → ④ 修复验证
- 核心：必须找到根因才能提出修复方案

**8. requesting-code-review（请求代码审查）**
- 触发：任务完成、主要特性完成、合并前
- 方法：派遣 `superpowers:code-reviewer` 子代理，提供设计文档、需求、git SHA
- 分类：Critical（阻塞进度）> Important（继续前先修）> Minor（稍后修）

**9. receiving-code-review（接收审查反馈）**
- 核心：技术评估，非表演性同意
- 流程：完整阅读 → 自己陈述需求 → 验证现有代码 → 评估正确性 → 响应或质证 → 实施
- 禁止："You're absolutely right!"、"Great point!"、盲目实施

**10. verification-before-completion（完成前验证）**
- 触发：任何成功声明前
- **铁律**：没有新鲜验证证据不能声称完成
- 流程：识别验证命令 → 运行完整命令 → 读全输出 → 检查退出码 → 核实输出 → 才能声称
- 禁止："should"、"probably"、"seems to"、信任代理报告

**11. dispatching-parallel-agents（并行代理调度）**
- 触发：3+ 独立失败可同时调查
- 使用场景：多个测试文件失败、不同子系统破损、无共享状态

**12. finishing-a-development-branch（完成开发分支）**
- 触发：所有任务完成，测试全过
- 流程：验证测试 → 确定基础分支 → 呈现 4 个选项 → 执行选择
- 选项：① 本地合并 ② 推送+创建 PR ③ 保持分支 ④ 丢弃

#### 阶段 5：文档与参考

**13. writing-skills（编写技能）**
- 目的：为 AI 代理记录可复用的技巧、模式、工具
- 本质：将 TDD 应用到流程文档
- 循环：压力测试（RED）→ 文档（GREEN）→ 找漏洞（REFACTOR）→ 验证

**14. using-superpowers（使用技能入门）**
- 目的：引导用户理解如何发现、调用技能
- 核心规则：任何 1% 可能应用的技能都必须调用
- 优先级：用户指令 > 技能 > 系统提示

### 3.2 技能文件结构

```
skills/
  skill-name/
    SKILL.md              # 必需：YAML 前缀 + 内容
    reference-file.md     # 可选：100+ 行重参考
    examples/             # 可选：代码示例
```

**SKILL.md 前缀（YAML）**：
```yaml
---
name: brainstorming
description: "Use WHEN... — 描述触发条件，不是功能"
---
```

---

## 四、工作流架构与设计

### 4.1 主工作流：从创意到完成

```
用户请求
    ↓
brainstorming [强制]
├─ 探索项目上下文
├─ 澄清问题（一次一个）
├─ 提议 2-3 种方案
├─ 呈现设计（按部分获批）
├─ 编写设计文档
├─ 自审查（占位符/一致性/范围/歧义）
└─ 用户审核（设计文档）
    ↓
using-git-worktrees [强制]
├─ 检测工作区位置
├─ 验证 .gitignore
├─ 创建隔离 worktree
├─ 运行项目设置
└─ 验证基线测试
    ↓
writing-plans [强制]
├─ 文件结构规划
├─ 咬度适当的任务（2-5 min 一个）
├─ 每步完整代码块
├─ 自审查（规范覆盖/占位符/类型）
└─ 提供执行选择
    ↓
选择执行方式
├─ subagent-driven-development [推荐]
│  ├─ 派遣实施子代理（per 任务）
│  ├─ 规范符合性审查
│  ├─ 代码质量审查
│  └─ 修复循环直至通过
│
└─ executing-plans [备选]
   ├─ 批量执行任务
   ├─ 验证每一步
   └─ 检查点审查
    ↓
finishing-a-development-branch [强制]
├─ 验证所有测试通过
├─ 呈现 4 个选项（合并/PR/保留/丢弃）
└─ 执行用户选择
    ↓
完成
```

### 4.2 测试驱动开发循环

```
RED → [看到失败] → GREEN → [所有通过] → REFACTOR → [保持绿色] → 完成
```

### 4.3 系统化调试流程

```
阶段 1: 根因调查
├─ 仔细读错误信息
├─ 一致复现
├─ 检查最近变化
└─ 多组件系统中添加诊断

阶段 2: 防御-深度
├─ 边界日志
├─ 数据流追踪
└─ 条件验证

阶段 3: 条件基础等待
└─ 等待特定条件而不盲目轮询

阶段 4: 修复验证
└─ 验证问题确实被修复
```

---

## 五、技术栈与依赖

### 5.1 项目配置

```json
{
  "name": "superpowers",
  "version": "5.0.6",
  "type": "module",
  "main": ".opencode/plugins/superpowers.js"
}
```

**关键特性**：
- **ES6 Modules**：使用 `"type": "module"` 声明
- **入口点**：OpenCode 插件实现
- **零依赖**：核心功能无外部 npm 依赖

### 5.2 支持的平台

| 平台 | 类型 | 集成方式 |
|------|------|---------|
| Claude Code | IDE 官方 | `/plugin install superpowers@claude-plugins-official` |
| Cursor | 编辑器 | `/add-plugin superpowers` |
| Google Codex | CLI | 手动配置 |
| OpenCode | 平台 | `opencode.json` plugin 数组 |
| Google Gemini CLI | CLI | `gemini extensions install` |

### 5.3 插件架构

**Claude Code/Cursor 插件配置**：
```json
{
  "skills": "./skills/",
  "agents": "./agents/",
  "commands": "./commands/",
  "hooks": "./hooks/hooks-cursor.json"
}
```

**OpenCode 插件实现**（`superpowers.js`）：
```javascript
export const SuperpowersPlugin = async ({ client, directory }) => {
  return {
    // 1. 动态注入 skills 目录到配置
    config: async (config) => {
      config.skills.paths.push(superpowersSkillsDir);
    },

    // 2. 系统提示转换注入 bootstrap 内容
    'experimental.chat.system.transform': async (input, output) => {
      output.system.push(bootstrapContext);
    }
  };
};
```

### 5.4 会话启动钩子

**触发**：`SessionStart` 事件（startup、clear、compact）

**脚本**：`hooks/session-start`（Bash）
- 检测平台（Claude Code vs Cursor vs 其他）
- 读取 `using-superpowers` 技能全文
- 转义为 JSON
- 注入上下文到会话

**目的**：每个新会话都自动加载使用说明

---

## 六、核心概念与设计原则

### 6.1 设计原则

**1. 一步一步（Step-by-Step）**
- 澄清问题：一次一个问题
- 呈现选项：多项选择优于开放式
- 增量验证：呈现 → 获批 → 继续

**2. 隔离与清晰（Isolation & Clarity）**
- 每个单元一个目的
- 良好定义的接口
- 可独立理解和测试

**3. 复杂性简化（Complexity Reduction）**
- 简洁为主要目标
- YAGNI：无情地移除不必要特性
- 小、专注的文件优于大文件

**4. 证据优先于断言（Evidence Over Claims）**
- 验证命令 → 看输出 → 才能声称结果
- 禁止："should work"、"probably"、"seems to"
- 代理完成报告 ≠ 实际验证

**5. 系统性优于临时（Systematic Over Ad-Hoc）**
- 流程优于猜测
- 文档化的步骤
- 可重复的方法

**6. TDD 作为基础**
- 编写失败的测试 → 看失败 → 最小代码 → 看通过 → 重构
- 应用于：代码、技能文档、调试流程

### 6.2 为什么选择 Skills 而不是 Prompt Injection？

- **可发现**：用户可列出、选择、理解可用技能
- **可版本化**：技能随插件更新，无版本冲突
- **可扩展**：用户可在 `~/.claude/skills` 添加自定义技能
- **可隔离**：技能按阶段/功能组织，易于找到相关的
- **可重复**：多项目、多会话间一致

### 6.3 为什么子代理优于同会话执行？

- **上下文保留**：控制器保留协调上下文
- **隔离失败**：一个子代理的问题不影响其他
- **新鲜头脑**：每个任务都由无历史偏见的代理处理
- **自动检查**：两阶段审查集成到流程
- **快速迭代**：子代理可快速问-答-实施-验证

---

## 七、配置文件详解

### 7.1 hooks.json（Claude Code）

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

- **matcher**：触发条件（`startup`、`clear`、`compact`）
- **async**: false = 同步执行（等待完成）
- **环境变量**：`${CLAUDE_PLUGIN_ROOT}` = 插件根目录

### 7.2 hooks-cursor.json 与 hooks.json 的差异

| 差异点 | Claude Code | Cursor |
|--------|-------------|--------|
| 字段名 | `SessionStart` | `sessionStart` |
| 版本字段 | 无 | `version: 1` |

---

## 八、测试与质量保障

### 8.1 测试结构

```
tests/
├── brainstorm-server/          # 可视化伴侣服务器测试
│   ├── server.test.js
│   └── ws-protocol.test.js
├── subagent-driven-dev/        # 子代理工作流测试
│   ├── go-fractals/            # 实战示例：Go 分形生成
│   └── svelte-todo/            # 实战示例：Svelte Todo 应用
├── explicit-skill-requests/    # 显式技能请求测试
│   └── prompts/
└── skill-triggering/           # 技能自动触发测试
    └── prompts/
```

### 8.2 测试方法

**1. 子代理压力测试**（技能开发用）：
- 派遣没有技能的子代理 → 观察偏差（RED）
- 编写技能文档（GREEN）
- 寻找漏洞，重复（REFACTOR）

**2. 工作流集成测试**：
- 实际项目执行（Go 分形、Svelte Todo）
- 验证端到端工作流
- 检查技能自动触发

---

## 九、版本演进与最新特性

### 9.1 v5.0.6（2026-03-24）— 当前版本

**关键变化**：
- **内联自审查替代子代理审查循环** — 规范和计划自审查用清单替代子代理分发，从约 25 分钟减少到约 30 秒，质量相当
  - brainstorming：占位符扫描 + 一致性 + 范围 + 歧义检查
  - writing-plans：规范覆盖 + 占位符 + 类型一致性
- **Brainstorm 服务器目录重构** — 内容和状态分离到同级子目录
  - `content/` — 服务到浏览器的 HTML
  - `state/` — 事件、pid、日志
- **Owner-PID 生命周期修复** — 跨用户 SSH、WSL 上的虚假关闭修复

### 9.2 v5.0.5（2026-03-17）

- **Brainstorm 服务器 ESM 修复** — Node.js 22+ 兼容性
- **Windows Owner-PID 修复** — MSYS2 PID 不可见处理
- **执行交付** — 恢复 subagent-driven vs executing-plans 的用户选择

### 9.3 v5.0.4（2026-03-16）

- **单次整体计划审查** — 替代逐块审查，减少 token
- **更高的阻塞门槛** — 校准审查器，仅阻塞真实问题
- **OpenCode 改进** — 一行安装，无需符号链接

---

## 十、架构优势与设计智慧

### 10.1 为什么这个设计有效？

**1. 约束强化卓越**
- 硬门控（HARD-GATE）：设计必须在实施前批准
- 禁止列表（Forbidden）：明确禁止不佳做法而非仅建议

**2. 上下文隔离**
- worktrees：特性分支隔离
- 子代理：会话隔离，无历史污染
- 技能：流程隔离，清晰边界

**3. 检查点与验证**
- 多层审查（规范 → 代码质量）
- 证据优先（看输出，不信任报告）
- 测试循环（RED → GREEN → REFACTOR）

**4. 渐进与协作**
- 一次一个问题
- 增量验证
- 设计批准 → 计划 → 执行 → 完成

**5. 可复用的知识**
- 技能库：模式一旦写好，永远可用
- 文档化过程：一致重复
- TDD 应用于所有层级（代码、文档、流程）

---

## 十一、常见陷阱与最佳实践

### 11.1 常见陷阱

| 陷阱 | 原因 | 修复 |
|------|------|------|
| "太简单，跳过设计" | 简单项目更需要明确 | 遵循流程，即使设计短 |
| 盲目相信 AI "完成" | 报告 ≠ 验证 | 运行验证命令，看输出 |
| "just this once" 违反 TDD | 时间压力的诱惑 | 记住：TDD 总是更快 |
| 部分实施代码审查反馈 | 项目相关的反馈 | 澄清所有项后再实施 |
| 使用旧工作流跳过 worktree | 遗留习惯 | worktree = 强制，无例外 |

### 11.2 使用最佳实践

**1. 尊重工作流顺序**：
```
brainstorming → using-git-worktrees → writing-plans →
  subagent-driven-development → verification → finishing-a-development-branch
```

**2. 编写完整的设计文档** — 即使"简单"项目，2-3 句也可以

**3. 优先子代理驱动开发** — 更高质量、更快速度

**4. 验证一切** — 看到失败的测试、看到通过的验证，不要相信 AI 报告

**5. 利用技能库** — 定期查看可用技能，为常见模式创建自定义技能

---

## 十二、已知限制

| 限制 | 原因 | 解决方案 |
|------|------|---------|
| Windows Owner-PID 监控禁用 | PID 命名空间不可见 | 依赖 30 分钟空闲超时 |
| 可视化伴侣在某些网络上失败 | 安全策略 | 使用 127.0.0.1 |
| Bash 5.3+ heredoc 挂起 | Bash 回归 | 迁移到 printf（已修复） |

---

## 十三、总结

### 项目核心特征

1. **工作流框架**：从创意到部署的完整、结构化的流程
2. **技能库**：14 个可复用、文档化的软件工程最佳实践
3. **多平台**：Claude Code、Cursor、Codex、OpenCode 支持
4. **子代理原生**：为 AI 协作编程优化
5. **TDD-首先**：测试驱动贯穿所有层（代码、文档、流程）
6. **零依赖**：核心逻辑无外部依赖，轻量级实现
7. **可扩展**：用户可添加自定义技能

### 项目成熟度

- **当前版本**：v5.0.6（2026-03-24）
- **活跃开发**：频繁更新和修复
- **社区**：Discord、GitHub Issues
- **文档**：详尽的设计文档和发布说明
- **生产就绪**：已在多个项目中验证

---

## 附录：常见文件路径参考

```
核心入口:
  package.json
  README.md

插件配置:
  .claude-plugin/plugin.json
  .cursor-plugin/plugin.json
  .opencode/plugins/superpowers.js

技能:
  skills/{skill-name}/SKILL.md
  skills/{skill-name}/*.md （支持文件）

配置:
  hooks/hooks.json （Claude Code）
  hooks/hooks-cursor.json （Cursor）
  hooks/session-start （启动脚本）

代理:
  agents/code-reviewer.md

文档:
  docs/README.codex.md
  docs/README.opencode.md
  docs/superpowers/specs/ （设计示例）
  docs/superpowers/plans/ （计划示例）

测试:
  tests/brainstorm-server/
  tests/subagent-driven-dev/
  tests/skill-triggering/
```
