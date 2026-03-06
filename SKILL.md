---
name: web-autopilot
description: >
  Record any web app operation once, AI turns it into a reusable automation tool.
  Use when: (1) automating repetitive tasks on any web application (reports, submissions,
  data extraction), (2) creating no-code automation for any logged-in web app,
  (3) building callable tools from recorded browser sessions.
  Supports REST, GraphQL, form submissions, file uploads, any login method.
  Task types: query/export (data extraction) and submit (form submissions like expense reports,
  travel requests, payment requests).
---

# Web Autopilot

> 在任何 Web 应用上操作一遍，AI 帮你变成自动驾驶。

## Overview

Record → Analyze → **Confirm Fields** → Generate → Test → Register as Tool

```
🎬 Record         用户在浏览器操作一遍（登录后即可录制）
🔍 Analyze        AI 分析网络流量，区分固定/动态/会话字段
✅ Confirm Fields  【提交类必须】向用户确认字段分类，确保准确
📝 Generate       生成可复用的 TS 脚本 + 字段映射
🧪 Test           循环测试脚本，最多 5 轮自动修复
🔧 Register       注册为 OpenClaw tool，下次直接调用
```

## Task Types

### 📊 Query / Export（查询/导出类）
数据抓取、报表导出。脚本跑完自动输出结果，无需人工干预。
例：拉取销售报表、抓取服务项目数据、抓取 AC 收入数据

### 📝 Submit（提交类）
提交申请单、报销单、付款单等。**每次执行需要传入动态参数**。
例：提交出差申请、提交报销申请、提交付款申请

提交类的关键挑战：**正确区分哪些字段固定、哪些字段每次都变**，并在生成脚本前与用户确认。

## Skill Directory

```
~/.openclaw/rpa/
├── recordings/<task-name>/recording.json
├── tasks/<task-name>/
│   ├── task-meta.json
│   ├── run.ts
│   └── field-mapping.json
└── sessions/<domain>.session.json
```

Skill scripts: `/opt/homebrew/lib/node_modules/openclaw/skills/web-autopilot/scripts/`

---

## Commands

### 1. `record` — Record a workflow

Ask user: task name, login URL or app URL.

```bash
cd /opt/homebrew/lib/node_modules/openclaw/skills/web-autopilot

# Option A: Start from login page (SSO, OAuth, username/password, etc.)
npx ts-node scripts/record.ts --name "任务名" --sso-url "https://login.example.com"

# Option B: Start directly from app (if already logged in or no login needed)
npx ts-node scripts/record.ts --name "任务名" --app-url "https://app.example.com"
```

Run in PTY mode (pty: true, background: true). User operates browser, types "done" when finished.

Note: `--sso-url` is a legacy parameter name; it works for **any** login URL (SSO, OAuth, plain login page, etc.).

### 2. `analyze` — Analyze the recording (AI does this)

Read recording.json, separate login traffic from business traffic, identify core APIs.

**Key steps:**
1. Read `~/.openclaw/rpa/recordings/<task>/summary.txt` for overview
2. Parse recording.json to extract all API calls to app domain
3. For each POST/PUT/PATCH with meaningful body:
   - Classify fields: FIXED / DYNAMIC / SESSION / RELATIONAL
   - Detect protocol: rest-json / graphql / form-urlencoded / multipart
4. Map the complete API sequence (prerequisites → main operation → follow-ups)
5. **Analyze ALL response fields and create field-mapping.json with Chinese names**
6. Create task-meta.json
7. **【提交类】完成分析后，必须向用户展示字段分类确认表（见下方）**

#### Field Classification（字段分类）

| 类型 | 含义 | 处理方式 |
|------|------|----------|
| **FIXED** | 每次提交都相同（审批流ID、公司主体、币种、费用类型枚举…） | 硬编码进脚本 |
| **DYNAMIC** | 每次提交都不同（金额、日期、事由、附件路径…） | 变成命令行 `--参数` |
| **SESSION** | 登录态 token/cookie，自动管理 | 由 session.ts 注入 |
| **RELATIONAL** | 需要先查另一个接口才能得到的 ID（如项目ID、人员ID…） | 脚本内自动查询，或作为 DYNAMIC 参数 |

#### Field Analysis Rules (MANDATORY)

**Every field must have a Chinese label.** Including system-generated field names.

推断优先级：
1. **数据值类型**：时间戳(10^12-13) / 金额(看上下文) / 枚举(固定值) / URL / JSON对象
2. **字段名模式**：*time/*date/*_at → 时间 | *amount/*price/*cost → 金额 | *id/*_key → ID | *status/*state → 状态
3. **业务上下文**：查看关联字段、API 端点名推断
4. 无法确定 → 标注 `(未知含义:示例值)`

#### 【提交类专属】字段确认步骤（MANDATORY for Submit tasks）

分析完成后，**必须向用户展示以下确认表，等待用户确认后再生成脚本**：

```
📋 字段分类确认 — <任务名>

✅ FIXED（固定，硬编码）：
  - approvalFlowId: "xxx"  → 审批流ID
  - companyId: "yyy"       → 公司主体
  - currency: "CNY"        → 币种

🔄 DYNAMIC（每次变化，作为参数传入）：
  - amount          → 金额（示例: --amount 1500）
  - startDate       → 出发日期（示例: --startDate 2026-03-10）
  - endDate         → 返回日期（示例: --endDate 2026-03-12）
  - destination     → 目的地（示例: --destination 北京）
  - reason          → 出差事由（示例: --reason "客户拜访"）
  - attachments     → 附件路径（示例: --attachments ~/Desktop/receipt.jpg）

🔗 RELATIONAL（自动查询）：
  - projectId       → 关联项目ID（通过项目名称自动查询，--projectName "XX项目"）

❓ 待确认（AI 无法确定）：
  - field_abc123    → 含义不明（录制时值为 "0"），建议: FIXED("0") 还是 DYNAMIC?

请确认以上分类是否正确，或指出需要调整的字段。
```

**用户确认后才能进入 Generate 步骤。**

#### CSV Export Rules (MANDATORY)

- **保留所有字段**，包括隐藏字段、动态字段、系统字段，不能裁剪
- 字段顺序：从数据中收集的原始顺序，**绝对不能排序**（排序会导致列错位）
- JSON/对象字段 → 转 JSON 字符串存储
- 使用 `csv.writer` + 正确引号处理，避免 JSON 内逗号导致列错位

### 3. `generate` — Generate the task script

**生成前检查清单（查询/导出类）**：
- ✅ 所有字段都在 field-mapping.json 中
- ✅ 所有字段都有中文标注
- ✅ CSV 导出使用 field-mapping.json 映射列名
- ✅ 字段顺序保留原始顺序

**生成前检查清单（提交类）**：
- ✅ 用户已确认字段分类（FIXED / DYNAMIC / RELATIONAL）
- ✅ 所有 DYNAMIC 字段已转为命令行参数（含类型、示例值、是否必填）
- ✅ RELATIONAL 字段有自动查询逻辑或对应参数
- ✅ 脚本有 `--dry-run` 模式（打印请求体但不实际提交，供测试用）
- ✅ 脚本输出提交结果（成功/失败 + 单据号/链接）

**提交类脚本调用示例**（生成后写入 task-meta.json 的 usage 字段）：
```bash
# 预览（不实际提交）
npx ts-node run.ts --dry-run --amount 1500 --startDate 2026-03-10 ...

# 正式提交
npx ts-node run.ts --amount 1500 --startDate 2026-03-10 --destination 北京 --reason "客户拜访"
```

### 4. `test` — Iterative test loop (max 5 rounds)

Run script → check output → if error: diagnose → fix → repeat.

| Error | Cause | Fix |
|-------|-------|-----|
| 401/403 | Session expired / wrong auth | Re-check auth headers, re-login |
| 400 | Wrong field name/type | Compare with recording |
| 404 | Wrong URL | Check URL exactly |
| JSON parse error | Response is HTML | Log resp.raw |

### 5. `run` — Execute a registered task

```bash
npx ts-node ~/.openclaw/rpa/tasks/<task>/run.ts --param1 value1
```

### 6. `list` — List all tasks

```bash
npx ts-node /opt/homebrew/lib/node_modules/openclaw/skills/web-autopilot/scripts/run-task.ts --list
```

---

## Session & Credential Management

### Session（Cookie/Token 存储）

Sessions are cookie-based and work with **any login method**:
- SSO (OIDC, SAML, CAS, etc.)
- OAuth / OAuth2
- Username + password forms
- Any browser-based authentication

Session files: `~/.openclaw/rpa/sessions/<domain>.session.json`

### Credentials（加密凭证存储）

Login credentials are stored **encrypted** (AES-256-GCM) in a separate file, **never明文存储**。

File: `~/.openclaw/rpa/credentials.enc`
- Encryption key = machine identity (hostname+username) + optional `RPA_CREDENTIAL_KEY` env var
- File permissions: 0600 (owner only)
- 支持从 recording.json 中自动提取并加密保存

```bash
# 管理凭证
npx ts-node scripts/utils/credentials.ts list                    # 列出已保存的域名+用户名
npx ts-node scripts/utils/credentials.ts save <domain> <user> <pass>  # 手动保存
npx ts-node scripts/utils/credentials.ts delete <domain>         # 删除
npx ts-node scripts/utils/credentials.ts extract <recording.json> # 从录制文件提取
```

### Auto-Login Flow（自动登录）

当 session 过期时，自动登录流程：

```
1. 读取 credentials.enc 中对应域名的加密凭证
2. 根据 loginFlow.type 选择登录策略
3. 启动浏览器（有凭证则 headless，无凭证则 headed）
4. 执行登录步骤 → 跟随重定向 → 到达目标应用
5. 捕获 cookies/tokens → 保存新 session
6. 全部失败 → 打开有头浏览器让用户手动登录（兜底）
```

#### Login Flow Types（登录流类型）

生成脚本时，**必须根据录制识别登录类型**，写入 `task-meta.json` 的 `loginFlow` 字段：

| type | 场景 | 自动登录方式 | 示例 |
|------|------|-------------|------|
| **`api`** | SSO/应用提供 REST 登录接口，一次 POST 完成认证 | 直接调 API → 跟随重定向 | 企业 SSO (`POST /api/sso/login`) |
| **`form`** | 单页面登录表单（username + password 在同一页） | 填充表单字段 → 点击提交 | 常见后台管理系统 |
| **`multi-step`** | 多步骤登录（邮箱→下一页→密码→下一页→可能有 2FA） | 按步骤序列执行 | Google, Microsoft, Okta |
| **`manual-only`** | 有验证码/2FA/风控，无法全自动 | 直接打开 headed 浏览器 | 银行系统、强验证码网站 |

#### loginFlow Schema（task-meta.json）

```jsonc
{
  "loginFlow": {
    "type": "api",              // api | form | multi-step | manual-only
    "loginUrl": "https://sso.example.com",
    "loginDomain": "sso.example.com",
    "appDomain": "app.example.com",

    // ── type=api 专用字段 ──
    "loginApiPath": "/api/sso/login",
    "authType": "webLocalAuth",       // 可选，API body 中的认证类型字段
    "appId": "6943592991949180",      // 可选，SSO 门户的应用 ID（用于 forward 跳转）
    "appForwardUrl": "...",           // 可选，直接跳转 URL（替代 appId）

    // ── type=form 专用字段 ──
    "usernameSelector": "input[name='email']",    // 可选，自定义选择器
    "passwordSelector": "input[type='password']",
    "submitSelector": "button[type='submit']",

    // ── type=multi-step 专用字段 ──
    "steps": [
      { "action": "fill", "selector": "input[type=email]", "field": "username" },
      { "action": "click", "selector": "#identifierNext" },
      { "action": "wait", "selector": "input[type=password]", "timeoutMs": 5000 },
      { "action": "fill", "selector": "input[type=password]", "field": "password" },
      { "action": "click", "selector": "#passwordNext" }
    ],

    // ── 通用字段 ──
    "successIndicator": "url_contains:app.example.com",  // 判断登录成功的条件
    "postLoginWaitMs": 3000          // 登录成功后等待时间（等 cookie 落地）
  }
}
```

#### Analyze 阶段登录识别指南（MANDATORY）

在 analyze 步骤中 **必须** 完成以下登录分析：

1. **提取凭证** → `credentials.ts extract <recording.json>`（自动识别 POST body 中的 username/password）
2. **识别登录类型** → 检查 recording 中的登录流：
   - 有明确的 `POST login/auth API` → type = `api`
   - 有表单填充 actions（password type input）且在同一页 → type = `form`
   - 有多次表单填充 actions，中间夹着页面导航 → type = `multi-step`
   - 有验证码图片请求、reCAPTCHA 脚本 → type = `manual-only`
3. **记录 SSO → 应用的跳转路径**：
   - 是否通过 appId forward？
   - 是否通过 redirect_uri 回调？
   - Token 在 URL query / response body / cookie 中的哪里？
4. **写入 loginFlow** → 所有字段写入 `task-meta.json`
5. **脱敏** → 将 recording.json 中的密码替换为 `[REDACTED]`

**⚠️ 如果 credentials.ts extract 无法提取（如 Google 多步骤登录），需要提示用户手动保存凭证：**
```bash
npx ts-node scripts/utils/credentials.ts save accounts.google.com user@gmail.com 'password'
```

#### 生成脚本时的登录代码模板

根据 `loginFlow.type` 选择对应的自动登录实现：

**type=api（REST API 登录）：**
```typescript
// API 登录 → 跟随重定向 → 跳转到应用
const resp = await page.evaluate(async (p) => {
  const r = await fetch(p.url, { method: 'POST', headers: {'Content-Type':'application/json'},
    body: JSON.stringify(p.body), credentials: 'include' });
  return { status: r.status, ok: r.ok };
}, { url: loginApiUrl, body: { authType, credential: { username, password } } });
```

**type=form：**
```typescript
await page.fill(loginFlow.usernameSelector || 'input[name="username"]', cred.username);
await page.fill(loginFlow.passwordSelector || 'input[type="password"]', cred.password);
await page.click(loginFlow.submitSelector || 'button[type="submit"]');
```

**type=multi-step：**
```typescript
for (const step of loginFlow.steps) {
  if (step.action === 'fill') {
    const value = step.field === 'username' ? cred.username : cred.password;
    await page.fill(step.selector, value);
  } else if (step.action === 'click') {
    await page.click(step.selector);
  } else if (step.action === 'wait') {
    await page.waitForSelector(step.selector, { timeout: step.timeoutMs || 10000 });
  }
}
```

**type=manual-only：**
```typescript
// 直接打开 headed 浏览器，等用户手动完成
const browser = await pw.chromium.launch({ headless: false });
// ... wait for successIndicator
```

**task-meta.json loginFlow 示例（SSO → 企业应用）：**
```json
{
  "loginFlow": {
    "type": "api",
    "loginUrl": "https://para.sso360.cn",
    "loginDomain": "para.sso360.cn",
    "loginApiPath": "/api/sso/login",
    "authType": "webLocalAuth",
    "appId": "6943592991949180",
    "appDomain": "app.ekuaibao.com",
    "successIndicator": "url_contains:app.ekuaibao.com"
  }
}
```

### ⚠️ Security Rules（安全规则，MANDATORY）

1. **recording.json 中的密码必须在 analyze 完成后立即脱敏**（替换为 `[REDACTED]`）
2. **credentials.enc 是加密的二进制文件**，不要尝试直接读取或编辑
3. **credentials.enc 和 sessions/ 目录绝对不能纳入版本控制或分享**
4. **Skill 安装包(.skill)中不包含任何凭证、session、recording 数据**
5. **生成的 task 脚本(run.ts)中不能硬编码任何密码**

---

## Known Issues & Lessons Learned

### 🔐 Login flow must match app — don't assume one-size-fits-all
- 当前已实现的脚本使用 `type=api` 模式（企业 SSO → 业务应用）
- **每个新应用录制时必须重新识别登录类型**，不能复用旧脚本的登录逻辑
- Google/Microsoft 等多步骤登录需要 `type=multi-step` + steps 序列
- 有验证码/2FA 的系统只能 `type=manual-only`
- 将登录逻辑内联到 run.ts 中（而非 import 外部 login.ts）更稳定，因为 Node ESM/CJS 兼容性问题

### ⚠️ Node v25 ESM 兼容性
- Node v25 默认 ESM，`require()` 不可用
- 解决方案：在 task 目录下放 `tsconfig.json` 强制 `"module": "commonjs"`
- Playwright 等依赖需要用完整路径 require：`require('/opt/.../node_modules/playwright')`
- 跨目录 import .ts 文件在 ts-node 下不稳定，推荐将关键逻辑内联到 run.ts

### ⚠️ Multi-tab traffic capture (fixed)
Some login flows or apps open new tabs. Recorder uses `context.on('request/response')` to capture ALL tabs.

### 📋 CSV must include ALL fields with Chinese labels
- Never crop fields — include everything from the API response
- System-generated field names (e.g. `field_*`, `attr_*`, `custom_*`) must be analyzed from sample data
- Create field-mapping.json for every task
- Field order: preserve original order from data, **never sort**
- Use proper CSV quoting to handle JSON fields with commas

### 📝 Submit tasks: always confirm field classification before generating
- Never skip the field confirmation step — wrong FIXED/DYNAMIC split breaks every future submission
- Fields that look fixed (e.g. a hardcoded project ID) might actually need to be dynamic in real use
- Always include `--dry-run` in generated scripts so users can verify the request body before committing
- RELATIONAL fields (e.g. approver ID looked up by name) should be auto-resolved in script, exposed as human-readable params

---

## File Locations

| Item | Path |
|------|------|
| Recorder | `scripts/record.ts` |
| Task runner | `scripts/run-task.ts` |
| Session utility | `scripts/utils/session.ts` |
| Login helper | `scripts/utils/login.ts` |
| Recordings | `~/.openclaw/rpa/recordings/<task>/` |
| Generated tasks | `~/.openclaw/rpa/tasks/<task>/` |
| Sessions | `~/.openclaw/rpa/sessions/<domain>.session.json` |
