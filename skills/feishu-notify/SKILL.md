---
name: feishu-notify
description: Send execution results, alerts, and summaries to Feishu (Lark) via webhook. Use this skill whenever a task completes, errors out, or needs to push a notification to Feishu. Also use when the user asks to "notify Feishu", "send to Feishu", "push results to Lark", or mentions needing to integrate with Feishu bots.
---

# Feishu Notification Skill

Send messages to Feishu (Lark) groups via bot webhook. Uses Feishu's interactive card format for clean, readable notifications.

## Quick Start

After completing your main task, send a notification:

**正确调用方式（强制使用文件路径作为第3参数）：**
```bash
WEBHOOK_URL="https://open.feishu.cn/open-apis/bot/v2/hook/c0e603a3-24b4-4629-a339-79c6f45a6adb"
# 1. 写入消息内容到文件
cat > feishu_msg.txt <<'EOF'
Your markdown message here.
Keep it concise and structured.
EOF
# 2. 文件路径作为第3参数调用（脚本会读取文件内容发送）
bash ~/.claude/skills/feishu-notify/scripts/send_to_feishu.sh "$WEBHOOK_URL" "Task Complete" feishu_msg.txt
# 3. 发送后清理临时文件
rm feishu_msg.txt
```

**⚠️ 禁止的错误调用方式：**
```bash
# ❌ 错误：通过stdin传入内容（在Windows/Multica环境下不可靠）
echo "message" | bash ~/.claude/skills/feishu-notify/scripts/send_to_feishu.sh "$WEBHOOK_URL" "Title"

# ❌ 错误：把文件路径当内容传（飞书会收到路径字符串而非文件内容）
bash ~/.claude/skills/feishu-notify/scripts/send_to_feishu.sh "$WEBHOOK_URL" "Title" <<< "$(pwd)/feishu_msg.txt"
```

**Windows (PowerShell):**
```powershell
$webhook = "https://open.feishu.cn/open-apis/bot/v2/hook/c0e603a3-24b4-4629-a339-79c6f45a6adb"
# 1. 写入消息内容到文件
Set-Content -Path feishu_msg.txt -Value "Your message here" -Encoding UTF8
# 2. 用文件路径作为第3参数调用
bash ~/.claude/skills/feishu-notify/scripts/send_to_feishu.sh $webhook "Task Complete" feishu_msg.txt
# 3. 清理
Remove-Item feishu_msg.txt
```

## Webhook URL

The default webhook URL is stored in `webhook.conf`. Each agent/team can use their own URL — pass it directly to the script rather than hardcoding.

**Current webhook:** `https://open.feishu.cn/open-apis/bot/v2/hook/c0e603a3-24b4-4629-a339-79c6f45a6adb`

## When to Notify

Send a Feishu notification when:
1. **Task completed** — summary of what was done, key findings
2. **Error occurred** — what failed, what's needed to fix it
3. **Scheduled report** — daily/weekly summaries (e.g., market reports)
4. **User explicitly asks** — any request to notify/push to Feishu

## Message Format Guidelines

- Use **card title** for the category + date (e.g., "财经新闻摘要：2026-06-17")
- Use **markdown body** for the content — Feishu cards support markdown
- Keep total message under 3000 characters to avoid truncation
- Put the most important info in the first 3 lines
- For long reports, send an executive summary with a note that full content is available on the platform

## Scripts

- `scripts/send_to_feishu.sh` — bash script, works on Linux/Mac/Git Bash
- `scripts/send_to_feishu.ps1` — PowerShell script, works on Windows

Both scripts accept: `<webhook-url> <title> [message-file]` (or stdin for message body).

## Integration Pattern

When building a scheduled or automated agent, add this at the end of the execution flow:

```
Task execution → Build summary → Call send_to_feishu.sh → Confirm sent
```

Do NOT send notifications for trivial operations (typo fixes, formatting). Notifications should be meaningful — the user sees them in their Feishu chat.
