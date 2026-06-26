---
name: nuwax-skill-publish
description: >
   将本地技能目录打包为zip并上传到棱镜Nuwax平台。触发词："打包上传技能"、"发布技能"、"更新平台技能"、"上传技能到棱镜"、"打包技能"、"publish skill"、"upload skill"、"部署技能"。
   当用户要求将技能发布/上传/更新到棱镜平台时必须调用本技能。
metadata:
   requires:
      mcp: ["chrome-devtools-mcp"]
      bins: ["7z"]
---

# 技能打包与上传棱镜平台

7-zip 打包 + Chrome DevTools MCP 操作棱镜平台。平台：`https://aios.topprismdata.com/space/6/skill-manage`

---

## 路由决策（最先执行）

选择技能后，**检查变更范围**决定走「快速编辑」还是「完整上传」：

1. 读取本地 `skills/<名>/SKILL.md`
2. 导航 `https://aios.topprismdata.com/space/6/skill-manage` → `wait_for` `["技能管理", "登录"]`
3. 搜索框输入技能名 → `wait_for` 技能名
4. 点击技能名进入详情页 → `wait_for` `["SKILL.md", "发 布"]`

现在检查页面上的「已发布」状态和本地变更：

- **只有 SKILL.md 有变更** → 走 **快速编辑**（下面的 Phase A）
- **多个文件变更 / 新增文件 / 技能不存在** → 走 **完整上传**（下面的 Phase B）

> 判断"只有 SKILL.md"的方法：如果用户只是改了 SKILL.md 里的文字（描述、触发词等），没有动 scripts/ 下的代码，就是只有 SKILL.md 变更。

---

## 约束

- 7z 路径：`C:/Program Files/7-Zip/7z.exe`
- 浏览器必须已登录，不自动执行登录
- 技能结束后不关闭浏览器
- zip ≤ 20MB
- 排除：`__pycache__/`、`*.pyc`、`*.db`、`node_modules/`、`.git/`、`.env`、`config.json`

---

## Phase A：快速编辑（~400 token，仅 SKILL.md 变更时使用）

> 直接在平台编辑器中更新 SKILL.md 内容，无需打包和上传。

1. 确认已进入技能详情页，且页面显示「SKILL.md」已选中（代码视图）
2. 如果不在代码视图：点击「代码」radio（`radiogroup` 中的「代码」选项）
3. **全选并替换编辑器内容**：
   - 使用 `mcp__chrome_tools__evaluate_script` 直接设置 Monaco 编辑器内容：
   ```js
   () => {
     const editor = document.querySelector('.monaco-editor');
     if (editor && editor.__vue__) return 'vue';
     // Fallback: find the textarea
     const ta = document.querySelector('textarea[aria-label="Editor content"]');
     if (ta) { ta.focus(); ta.select(); return 'textarea'; }
     return 'not found';
   }
   ```
   - 如果找到 editor：用 `press_key` `Control+a` 全选 → `type_text` 粘贴新的 SKILL.md 内容
   
   **关键**：由于编辑器内容较大，使用 `type_text` 逐字符输入不现实。改为：
   - `fill` 编辑器 textarea（uid 为 `Editor content` 的 textbox）—— 但 multiline markdown 可能截断
   - **推荐方案**：使用 `evaluate_script` 直接设置编辑器内容：
   ```js
   () => {
     // Monaco Editor
     const model = document.querySelector('.monaco-editor')?.__vue__?.$data;
     // Try global monaco API
     if (window.monaco?.editor?.getModels) {
       const models = window.monaco.editor.getModels();
       if (models.length > 0) {
         models[0].setValue(`CONTENT_PLACEHOLDER`);
         return 'monaco-ok';
       }
     }
     // Fallback: textarea
     const ta = document.querySelector('textarea[aria-label="Editor content"]');
     if (ta) {
       const nativeTextArea = ta.querySelector('textarea') || ta;
       nativeTextArea.value = `CONTENT_PLACEHOLDER`;
       nativeTextArea.dispatchEvent(new Event('input', {bubbles: true}));
       return 'textarea-ok';
     }
     return 'not-found';
   }
   ```
   
   **将本地 SKILL.md 内容嵌入脚本中**（替换 `CONTENT_PLACEHOLDER`），使用模板字面量转义反引号和 `$`。

4. `wait_for` `["有更新未发布"]` 确认平台检测到变更（编辑器内容变更后平台通常会自动标记）
5. 如果未出现「有更新未发布」，手动点击一下其他文件再点回 SKILL.md 触发变更检测
6. 进入 Phase C 发布

---

## Phase B：完整上传（~900 token，多文件变更或新建时使用）

### B1：打包

技能名映射到 `skills/<名>/`，不指定则列出 `skills/` 供选。

**关键**：必须从技能目录内部打包，确保 SKILL.md 在 zip 根目录。禁止从 `skills/` 父目录打包（会导致文件在子目录中，平台报「缺少必需的 SKILL.md 文件」）。

```bash
cd "skills/<名>" && "C:/Program Files/7-Zip/7z.exe" a -tzip "../<名>.zip" "." \
  -xr!"__pycache__/" -xr!"*.pyc" -xr!"*.db" -xr!"node_modules/" -xr!".git/" -xr!".env" -xr!"config.json"
```

> `"."` 而非 `"skills/<名>/"`，这样 SKILL.md 在 zip 根层级。

超 20MB → 警告停止。

### B2：登录检查（~300 token）

1. 导航 `mcp__chrome_tools__navigate_page` → 平台 URL
2. **用 wait_for 替代快照**：`mcp__chrome_tools__wait_for` 等待文本 `["技能管理", "登录"]`
   - 命中「技能管理」→ 已登录，进入 B3
   - 命中「登录」→ 停止，提示用户手动登录
   - **仅在 wait_for 失败/超时 30s 时才 fallback 到 take_snapshot**

### B3：查找与导入（~500 token）

搜索框输入技能名 `mcp__chrome_tools__fill`，然后 `wait_for` 目标技能名文本。

**A：同名技能已存在**

点击技能名标题 → 跳转详情页 → `wait_for` `["SKILL.md"]` 确认加载 → 三点菜单 →「导入技能」

**B：同名不存在**

「plus 技能」→「导入技能」

**C：上传**

1. 点击上传区域 → `upload_file`（zip 绝对路径）
2. 点击「确认导入」（可能需要重试一次，对话框保持打开状态则表示未生效，再点一次）
3. `wait_for` `["SKILL.md", "导入成功", "有更新未发布"]` 确认导入完成（「导入成功」toast 可能一闪而过，「有更新未发布」更可靠）
4. **必须 `press_key` `Escape`** — 导入成功后才 Escape，关闭可能残留的 Windows 原生文件选择器。**禁止在确认导入前按 Escape**，否则会关闭导入对话框导致上传失败

**D：导入成功后 → 清理 → 发布**

> **关键**：导入成功后先按 Escape 确保原生文件选择器已关闭，再点发布。残留选择器会遮挡发布按钮。

---

## Phase C：发布 — 红线规则（~500 token）

1. 点击「发 布」
2. `wait_for` `["系统广场", "发布技能"]` 打开发布对话框
3. 在 wait_for 返回的文本中检查「系统广场」checkbox 状态：
   - 含 **checked** → 点击取消 → **再次 confirm 已 unchecked**，未取消则中止
   - **不含 checked → 继续**
4. 确认「个人空间」勾选 +「系统广场」unchecked → 点击「确 定」
5. `wait_for` `["发布成功", "已发布"]` 确认

> **红线**：系统广场勾选状态下一律禁止点确定。必须先取消 → 二次确认 unchecked。

---

## Token 预算

| 模式 | Phase | 预估 token |
|------|-------|-----------|
| 快速编辑 | 路由 + 编辑 + 发布 | ~600 |
| 完整上传 | 打包 + 登录 + 导入 + 发布 | ~1,400 |

快速编辑比完整上传节省 ~60%，且无需等待 7z 打包。

---

## 错误处理

7z 不存在 → 提示安装；未登录 → 提示手动登录；zip 超 20MB → 警告；搜索不到 → 新建；上传/发布失败 → 提示检查。
快速编辑中 `evaluate_script` 返回 `not-found` → 降级到完整上传模式。
