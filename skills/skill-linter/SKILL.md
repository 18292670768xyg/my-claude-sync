---
name: skill-linter
description: >
  校验 Skill/Agent 编写质量的诊断工具。基于 5 维度 25 条规则，对 SKILL.md 执行深度校验并输出结构化报告。
  触发词：skill-lint、校验技能、检查 skill、lint skill、验证技能质量、
  技能质量检查、skill quality、review skill、技能评审、
  检查 SKILL.md、skill 诊断。Not for code review or architecture review.
---

# Skill Linter — 技能质量校验器

对 SKILL.md 执行基于规则的深度校验，输出结构化诊断报告：哪里好、哪里坏、该补什么、该删什么。

## When to use

- 写完新 Skill 后，运行校验确认质量
- 修改现有 Skill 后，运行回归检查
- 审查别人的 Skill 时，快速定位问题

## Workflow

### Step 1: 确定目标

读取用户指定的 Skill 路径，或自动检测当前对话上下文中的 SKILL.md。

验证目标存在：
```
Read <skill-path>/SKILL.md
```

如果路径不存在，询问用户指定。

### Step 2: 加载规则库

```
Read references/skill_rules.yaml
```

规则库包含 5 个维度、25 条规则，每条规则有 id、name、check_type、severity、fix。

### Step 3: 解析目标 Skill

收集以下信息：

| 数据 | 来源 | 方法 |
|------|------|------|
| frontmatter（name, description 等） | SKILL.md 前 `---` 之间 | 读取 YAML |
| body 行数 | SKILL.md frontmatter 之后 | 行计数 |
| 段落标题 | body 中的 `##` / `###` | 正则匹配 |
| 目录结构 | Skill 目录下所有文件 | Glob |
| references/ 文件行数 | 每个 reference 文件 | 行计数 |
| scripts/ 文件列表 | scripts/ 目录 | Glob |
| evals/ 是否存在 | evals/ 目录 | Glob |
| body 中引用的文件 | body 文本 | 搜索文件名 |

### Step 4: 逐条规则检查

按规则库顺序，对每条规则执行检查。

**确定性检查（check_type: deterministic）**：直接通过文本匹配/计数/正则判定。

| 规则 | 检查方法 |
|------|---------|
| TRIGGER-003 | 统计 description 字符数，判断是否在 50-500 |
| TRIGGER-004 | 检查 name 是否 kebab-case |
| STRUCT-001 | 统计 body 行数 |
| STRUCT-002 | 搜索 `##` 标题中的关键词 |
| STRUCT-005 | 搜索 Gotchas/陷阱/常见错误 标题及内容 |
| PROGRESS-001 | body > 400 行时检查 references/ 是否存在 |
| PROGRESS-002 | 列出 references/ 文件，在 body 中搜索每个文件名 |
| PROGRESS-003 | 检查 references/ 文件是否引用了更深层路径 |
| PROGRESS-004 | >300 行的 reference 文件是否含 TOC |
| PROGRESS-005 | 列出 scripts/ 文件，在 body 中搜索 |
| EVAL-001 | 检查 evals/ 目录或 body 中的测试段落 |
| EVAL-004 | 搜索模糊量词列表：适当/必要时/如果需要/尽可能/as appropriate/if necessary |
| RELIABLE-003 | 搜索裸相对路径模式 |

**语义检查（check_type: semantic）**：需要 LLM 理解内容后判定。

| 规则 | LLM 判定要点 |
|------|-------------|
| TRIGGER-001 | description 是否描述了何时使用，而非仅描述功能 |
| TRIGGER-002 | description 中有多少个具体触发词 |
| TRIGGER-005 | description 是否排除了 near-miss 场景 |
| STRUCT-003 | body 中是否存在大段背景叙述 |
| STRUCT-004 | body 中是否包含模型本来就知道的通用步骤 |
| RELIABLE-001 | 工具调用是否有错误处理 |
| RELIABLE-002 | 关键路径是否有验证步骤 |
| RELIABLE-004 | 是否有终止条件 |
| EVAL-002 | 输出格式是否可验证 |
| EVAL-003 | 工作流步骤是否可观测 |
| EVAL-005 | description 是否自相矛盾 |

语义检查时，将目标 Skill 的 frontmatter 和 body 提供给判定，结合规则定义做出 pass/fail 判断。

### Step 5: 计算分数

按规则库 scoring 部分的规则计算：

- 每条规则的满分 = dimension_weight / rule_count
- error 失败扣 100%，warning 扣 60%，info 扣 30%
- 维度得分 = 满分 - 扣分
- 总分 = 5 维度得分之和（满分 100）
- 等级：A(90+) / B(75+) / C(60+) / D(40+) / F(<40)

### Step 6: 输出报告

按以下格式输出：

```
═══════════════════════════════════════════
  Skill Lint Report: <name>
═══════════════════════════════════════════
总分: XX/100  等级: X

维度得分:
  路由与触发  XX/25  ████░░░░░░░░░
  内容与结构  XX/25  ████████░░░░░
  渐进披露    XX/20  ██████░░░░░░░
  防呆可靠    XX/15  ████░░░░░░░░░
  可评估性    XX/15  ███░░░░░░░░░░

❌ 失败 (必须修复):
  [RULE-ID] 规则名称
    证据: <具体发现的证据>
    修正: <fix 内容>
    来源: <规则来源说明>

⚠️ 警告 (强烈建议修复):
  [RULE-ID] 规则名称
    证据: <具体发现的证据>
    修正: <fix 内容>

💡 建议 (可选改进):
  [RULE-ID] 规则名称
    证据: <具体发现的证据>
    修正: <fix 内容>

➕ 建议补充:
  - <具体建议>

➖ 建议删除/精简:
  - <具体建议>

✅ 通过的规则:
  - [RULE-ID] 规则名称 — <一句话说明为什么通过>

═══════════════════════════════════════════
```

进度条绘制规则：每 10% 画一个字符，满分 = 13 个 `█`，不足部分用 `░`。

## Gotchas

- **不要修改目标 Skill**——本技能只诊断不治疗，所有建议由用户决定是否采纳
- **语义检查可能有假阳性**——LLM 判定不是 100% 准确，`⚠️` 级别的结论需要人工确认
- **EVAL-004 的模糊量词**：在代码示例中出现不算违规，只在指令性文本中才算
- **TRIGGER-002 触发词计数**：中英文都算，"架构评审"和"architecture review"各算 1 个
- **STRUCT-001 的 500 行**：计算的是 frontmatter 之后的 body，不是整个文件
- **先跑确定性检查再跑语义检查**——确定性检查的结果可以作为语义检查的输入

## Validation

本技能自身的质量标准：
- 对一个高质量 Skill 运行应得到 B+ 以上
- 对一个 description 模糊 + 无 gotchas + body 超长的 Skill 运行应得到 D 或 F
- 报告中每条失败规则必须有证据和修正建议
- 总分 = 5 维度分数之和

## References

- `references/skill_rules.yaml` — 25 条校验规则及评分标准
