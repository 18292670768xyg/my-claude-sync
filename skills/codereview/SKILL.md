---
name: codereview
description: "CodeReview Skill - 证据导向的代码审查工具，支持 assistant/gate 双模式、merge recommendation 和仓库级 waiver。使用 /cr 命令触发。核心目标：控制代码风险。"
compatibility: "Works with Cursor, Claude Code, OpenAI Codex, Trae, and other SKILL.md-compatible agents."
metadata:
  author: codereview
  version: "2.0.2"
---

# CodeReview Skill

CodeReview v2 是一个以证据为中心的代码审查 skill。它保留统一的 `/cr` 入口，但把审查拆成两种明确意图：
- `assistant`：提交前助手，用于尽早暴露明显风险。
- `gate`：分支级风险门禁，用于形成高置信度的合并建议。

当同一 workspace 下存在多个独立仓库（例如前端/后端/网关/SDK 分仓）时，CodeReview v2 还支持 `workspace` 联合 review：自动发现 changed repos，扩展 affected repos，并输出跨仓联合结论。

## 核心原则

- `mode`、`baseline`、`scope` 是三层独立语义，不能混在一起。
- 配置中的 mode、decision、waiver、recheck 策略必须驱动实际 workflow，而不是只作为装饰字段。
- `risk_score` 仍然保留，但仅作为摘要指标，不直接决定 gate 结论。
- `blocker` 必须同时满足配置中的严重性阈值、置信度阈值，并提供明确证据与阻塞理由。
- `waiver` 是仓库级、长期有效、可复用的风险记忆，默认保存在版本控制内。
- `workspace` 联合 review 的价值在于"系统一致性"，重点识别接口契约漂移、shared SDK/types 不一致、漏改仓与发布顺序风险。

## 快速开始

输入 `/cr` 即可触发代码审查：

```bash
# 提交前助手
/cr --assistant --staged

# 分支级风险门禁
/cr --gate --base main

# workspace 联合 review（自动发现 workspace 下有变更的仓库）
/cr --workspace
```

## 模式概览

| 模式 | 目标 | 默认 baseline | 典型输出 |
|-----|------|---------------|---------|
| `assistant` | 提前发现问题，减少返工 | 由 `review.modes.assistant.baseline` 决定 | 关键建议、摘要、辅助风险分数 |
| `gate` | 给合并决策提供高置信依据 | 由 `review.modes.gate.baseline` 决定 | `approve` / `caution` / `block`、blockers、waivers、recheck |
| `workspace` | 多仓联合判断系统一致性 | 每个仓库独立 baseline | workspace recommendation、Cross-Repo Findings、Per-Repo Findings |

## 工作流

### 1. 配置加载

首先加载配置文件 `codereview/config.yaml`（项目根目录下的全局配置）。如果配置文件不存在，系统 SHOULD创建**完整的 v2 配置模板**，而不是仅创建缩减骨架；新建模板至少应包含 `modes`、`collection`、`decision`、`waiver`、`recheck` 和 `output` 段。

**默认配置骨架：**
```yaml
review:
  model: "claude-3-5-sonnet-20241022"
  default_mode: "assistant"
  default_scope: "changed-symbols"
  risk_threshold: 70
  modes:
    assistant:
      baseline: "auto-local"
      adjudication: false
    gate:
      baseline: "base-branch"
      base_branch: "main"
      adjudication: true
  decision:
    blocker:
      min_severity: "high"
      min_confidence: "high"
      require_blocking_rationale: true
    recommendation:
      block_on_active_blockers: true
      caution_on_waived_blockers: true
      caution_on_should_fix: false
  waiver:
    enabled: true
    file: "codereview/waivers.yaml"
  recheck:
    enabled: true
    on_waiver_match: true
```

### 2. 选择 mode

**显式 mode 优先：**
- `--assistant` → 进入 `assistant`
- `--gate` → 进入 `gate`
- `--workspace` → 进入 `workspace` 联合 review
- `--base <branch>` 且未显式指定 mode → 默认进入 `gate`

**未显式指定 mode 时：**
- 使用 `review.default_mode`
- 如果命令显式提供与配置冲突的 mode 参数，命令参数覆盖配置

### 3. 选择 baseline 和 scope

`baseline` 回答"和谁比"，`scope` 回答"看哪些代码"。

**baseline 选项：**
- `working tree`
- `staged`
- `base...HEAD`

**scope 选项：**
- `changed-hunks`
- `changed-symbols`
- `files`
- `directories`
- `full-project`

未显式指定时，workflow SHALL使用 `review.default_scope` 以及 `review.modes.<mode>.baseline`。
默认情况下，优先收集 changed hunks，再向外扩展到所在 symbol、同文件依赖上下文，必要时少量补充跨文件上下文。

在 `workspace` 模式下：
- baseline MUST按仓库分别确定，并在联合报告中记录 `repo -> baseline` 元数据
- scope 默认为"各仓库的 changed hunks / changed symbols"，并允许在 affected repos 上执行只读的差异确认（用于漏改仓判断）

### 4. 代码收集与过滤

代码收集阶段 SHOULD遵循以下顺序：
1. 确定 baseline 对应的 diff
2. 识别 changed hunks 或显式路径
3. 向外扩展到函数、方法或类
4. 补充同文件上下文
5. 仅在必要时补充少量跨文件引用

配置中的 collection / ignore 规则 SHOULD在这一阶段生效，例如：
- 跳过 `CodeReview/`、`node_modules/`、`dist/` 等目录
- 跳过 lock 文件和明显的生成文件
- 对超大文件进行裁剪或忽略

在 `workspace` 模式下，collection 还 SHOULD支持两类扩展：
- **显式依赖（主）**：workspace 配置、package 依赖、服务注册、网关配置等
- **代码推断（补）**：import、SDK 调用、HTTP client、shared types 使用等

### 5. Candidate pass

所有模式都先执行 candidate pass，用于提高问题召回率。

提示词的核心目标是：
- 识别潜在问题
- 输出明确类别
- 尽量提供可验证的证据和触发条件
- 不要仅凭抽象最佳实践就下结论

### 6. Gate adjudication pass

`gate` 模式在 candidate pass 之后，还必须执行 adjudication pass。

adjudication pass 的目标是：
- 过滤低证据、低置信度的问题
- 要求每个 blocker 候选都补充证据、触发条件、影响范围和阻塞理由
- 按配置将问题归类为 `blocker`、`should-fix` 或 `note`

**blocker 判定原则：**
- `severity` 必须满足 `review.decision.blocker.min_severity`
- `confidence` 必须满足 `review.decision.blocker.min_confidence`
- `evidence` 足够具体
- 如果 `review.decision.blocker.require_blocking_rationale = true`，则 MUST提供 `blocking_rationale`

如果以上条件不成立，问题 SHOULD降级为 `should-fix` 或 `note`。

### 6.5 Workspace orchestration（联合 review 编排）

`workspace` 模式在保持 per-repo review 边界的基础上，额外执行：
- **repo discovery**：在 workspace 中发现独立仓库，并识别 `changed repos`
- **impact expansion**：扩展 `affected repos`，并区分 `explicitly_affected` 与 `inferred_affected`
- **cross-repo correlation**：识别接口契约漂移、shared SDK/types 不一致、漏改仓、发布顺序风险
- **workspace reporting**：输出 workspace recommendation，并分层展示 `Cross-Repo Findings` 与 `Per-Repo Findings`

### 7. Issue schema

v2 使用结构化 issue schema，而不是单一 `risk_level`：

```json
{
  "issues": [
    {
      "issue_id": "auth-log-token-leak",
      "file": "src/auth/logger.ts",
      "line": 42,
      "symbol": "logRequest",
      "category": "安全漏洞",
      "severity": "high|medium|low",
      "confidence": "high|medium|low",
      "disposition": "blocker|should-fix|note",
      "description": "问题描述",
      "evidence": "直接记录了 Authorization header",
      "trigger_condition": "当请求日志开启时",
      "impact": "可能泄露用户访问令牌",
      "blocking_rationale": "该问题会在合并后直接把敏感日志路径带入主分支，因此阻塞当前合并",
      "suggestion": "对敏感头做脱敏或避免记录"
    }
  ],
  "good_practices": [
    "做得好的地方"
  ],
  "risk_score": 42,
  "merge_recommendation": "approve|caution|block",
  "summary": "本次 review 的执行摘要",
  "recheck_results": [
    {
      "issue_id": "auth-log-token-leak",
      "previous_state": "waived",
      "current_state": "waived",
      "outcome": "still-open",
      "last_reviewed_at": "2026-03-26T10:20:00Z",
      "notes": "问题仍存在，但命中了仓库级 waiver"
    }
  ]
}
```

### 8. 稳定 `issue_id` 生成规则

`issue_id` SHOULD尽量稳定，不能只依赖行号。推荐组合：
- 归一化后的问题类型 key
- 相对文件路径
- `symbol` 路径或方法签名
- 附近代码锚点的 fingerprint

**不要** 把 `line` 当作唯一标识。行号可以漂移，`issue_id` 需要尽量跨多次 review 和跨分支复用。

### 9. Waiver 读取与匹配

CodeReview v2 默认读取 `codereview/waivers.yaml`（项目根目录下的全局配置）。

**waiver 规则：**
- waiver 是仓库级、纳入版本控制、长期有效的记录
- waiver 默认跨分支复用
- waiver SHOULD至少记录：`issue_id`、`reason`、`waived_by`、`created_at`
- 如果 gate run 命中已有 waiver，系统 SHOULD将该 issue 归类为 `Waived Issue`，而不是新的 `Active Blocker`
- gate run SHOULD更新匹配 waiver 的 `last_reviewed_at` 和 `last_recheck_outcome`
- 当前仓库级持久化仅覆盖**已存在 waiver 的 issue**；非 waiver 历史问题的 recheck 结果默认只体现在当次报告中，除非后续引入独立的 review-state 文件

### 10. Recheck 语义

`recheck` 用于表达"当前 gate run 如何重新评估历史已知风险"。在当前 v2 文档实现中，仓库级持久化范围仅限于 `waivers.yaml` 中已有的 issue。

当 `review.recheck.enabled = true` 时：
- 如果命中已有 waiver，系统 SHOULD生成一条 `recheck_results` 记录
- 如果某个**已存在 waiver 的 issue** 不再出现，系统 MAY生成 `outcome = not-observed` 或 `fixed`
- 如果某个**已存在 waiver 的 issue** 仍然存在但已降级，系统 SHOULD生成 `outcome = downgraded`
- 如果问题仍然存在并继续受 waiver 保护，系统 SHOULD生成 `outcome = still-open`

推荐的 `recheck_results` 字段包括：
- `issue_id`
- `previous_state`
- `current_state`
- `outcome`，取值为 `still-open|downgraded|fixed|not-observed`
- `last_reviewed_at`
- `notes`

### 11. Merge recommendation 与风险分数

`gate` 模式的 merge recommendation 应由 `review.decision.recommendation` 驱动：
- 如果 `block_on_active_blockers = true` 且存在未豁免的 `blocker` → `block`
- 如果 `caution_on_waived_blockers = true` 且命中了历史 waived blocker → `caution`
- 如果 `caution_on_should_fix = true` 且存在重要 should-fix → `caution`
- 否则 → `approve`

`risk_score` 仍然可以保留，但它只是摘要指标，用于提示总体风险密度，不再是 gate 主判据。

### 12. 报告生成

报告默认保存到 `CodeReview/YYYY-MM-DD-HHMMSS.md`。

**assistant 模式报告重点：**
- 执行摘要
- 关键发现
- `issue_id`、`severity`、`confidence`、`evidence` 等结构化字段
- `should-fix` 和 `notes`
- 做得好的地方

**gate 模式报告重点：**
- `Merge Recommendation`
- `Active Blockers`
- `Should Fix`
- `Notes`
- `Waived Issues`
- `Recheck Results`
- `Review Metadata`

如果 `review.output.include_risk_score = false`，则报告 SHOULD隐藏风险分数行，但不影响 gate 结论。

### 13. 向用户显示摘要

完成后向用户显示：
- 报告文件路径
- `mode`
- 关键问题数量
- `assistant` 模式下的摘要风险分数（若启用）
- `gate` 模式下的 merge recommendation、active blockers、waived issues 和 recheck 摘要

## 检查类别

| 类别 | 权重 | 说明 |
|-----|------|------|
| 语法/逻辑错误 | 3 | 语法错误、逻辑缺陷、边界条件处理 |
| 安全漏洞 | 5 | SQL 注入、XSS、认证授权问题、敏感数据泄露 |
| 性能问题 | 3 | N+1 查询、内存泄漏、算法效率、资源管理 |
| 代码风格/规范 | 1 | 命名规范、格式一致、代码可读性 |
| 可维护性 | 2 | 函数复杂度、模块划分、注释完整性 |
| 测试覆盖 | 2 | 测试缺失、测试质量、边界测试 |
| 架构设计 | 3 | 模块依赖、接口设计、扩展性、技术债务 |

## 配置文件

位置：`codereview/config.yaml`（项目根目录下的全局配置）

配置文件应覆盖以下能力：
- 默认 mode 和 baseline 策略
- collection / ignore 规则
- blocker 判定阈值
- recommendation 策略
- waiver 行为与路径
- recheck 行为
- 报告输出目录和是否显示风险分数
- workspace repo discovery、impact expansion 与联合报告策略

## 使用示例

```bash
# 默认使用 review.default_mode
/cr

# assistant 模式，自查暂存区变更
/cr --assistant --staged

# gate 模式，对比 main 分支
/cr --gate --base main

# gate 模式，指定目录并对比分支
/cr src/api/ --gate --base origin/main
```

## 输出文件

- Review 报告默认保存在 `CodeReview/`
- 仓库级 waiver 默认保存在 `codereview/waivers.yaml`

建议在 `.gitignore` 中添加：
```gitignore
CodeReview/
```

**不要** 将 `codereview/waivers.yaml` 加入 `.gitignore`。它是仓库级风险记忆的一部分，应该被提交和 review。
