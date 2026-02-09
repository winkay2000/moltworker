# Moltworker 完整設定指南

本文件記錄從零開始部署 Moltworker（OpenClaw on Cloudflare）的完整步驟，包含踩過的坑和解決方案。

---

## 目錄

1. [前置需求](#1-前置需求)
2. [Fork 與 Clone](#2-fork-與-clone)
3. [AI Provider 設定（Google Gemini）](#3-ai-provider-設定google-gemini)
4. [GitHub Actions 自動佈署](#4-github-actions-自動佈署)
5. [Cloudflare Secrets 設定](#5-cloudflare-secrets-設定)
6. [Cloudflare Access 設定](#6-cloudflare-access-設定)
7. [R2 持久化儲存設定](#7-r2-持久化儲存設定)
8. [Telegram 設定](#8-telegram-設定)
9. [首次啟動與配對](#9-首次啟動與配對)
10. [安全性設定](#10-安全性設定)
11. [Debug 與除錯](#11-debug-與除錯)
12. [踩坑紀錄](#12-踩坑紀錄)

---

## 1. 前置需求

- Cloudflare 帳號（免費方案即可）
- GitHub 帳號
- Google AI Studio API Key（免費）
- Node.js 22+
- Wrangler CLI：`npm install -g wrangler` 並執行 `wrangler login`

## 2. Fork 與 Clone

```bash
# Fork https://github.com/cloudflare/moltworker 到自己的帳號
git clone https://github.com/<你的帳號>/moltworker
cd moltworker
npm install
```

確認基本功能正常：
```bash
npm run typecheck
npm run lint
npm test
```

## 3. AI Provider 設定（Google Gemini）

本專案使用 Google Gemini 2.5 Flash 作為 AI 模型，透過 Google 原生 API（非 OpenAI 相容端點）。

### 取得 API Key

1. 到 [Google AI Studio](https://aistudio.google.com/) 登入
2. 點 **Get API Key** → **Create API key**
3. 複製 API Key

### 免費額度（實際觀察值，可能因帳號/地區而異）

| 限制 | 額度 |
|------|------|
| 每分鐘請求數 (RPM) | 5 |
| 每日請求數 (RPD) | 20 |
| 每分鐘 Token 數 (TPM) | 250,000 |

> **注意**：官方文件標示 RPM=10、RPD=500，但實測新帳號可能只有 RPM=5、RPD=20。如果不夠用可到 Google AI Studio 申請提升或啟用付費方案。

### 相關修改檔案

| 檔案 | 修改內容 |
|------|---------|
| `src/types.ts` | 新增 `GOOGLE_AI_STUDIO_API_KEY` 環境變數型別 |
| `src/gateway/env.ts` | 將 API Key 傳入 container |
| `src/index.ts` | 環境變數驗證加入 `GOOGLE_AI_STUDIO_API_KEY` |
| `start-openclaw.sh` | onboard 用 `--auth-choice gemini-api-key`；config patch 用原生 `google-generative-ai` API |
| `wrangler.jsonc` | 註解中新增說明 |

### 技術細節

**Onboard 階段**（首次啟動時建立 config）：
```bash
openclaw onboard --non-interactive --accept-risk \
    --auth-choice gemini-api-key \
    --gemini-api-key $GOOGLE_AI_STUDIO_API_KEY \
    --mode local \
    --gateway-port 18789 \
    --gateway-bind lan \
    --skip-channels \
    --skip-skills \
    --skip-health
```

**Config Patch 階段**（確保 provider 正確配置）：
```javascript
config.models.providers['google-gemini'] = {
    baseUrl: 'https://generativelanguage.googleapis.com/v1beta',
    apiKey: process.env.GOOGLE_AI_STUDIO_API_KEY,
    api: 'google-generative-ai',  // 原生 Google API，不是 openai-completions
    models: [{ id: 'gemini-2.5-flash', name: 'Gemini 2.5 Flash', contextWindow: 1048576, maxTokens: 8192 }],
};
config.agents.defaults.model = { primary: 'google-gemini/gemini-2.5-flash' };
```

## 4. GitHub Actions 自動佈署

### 建立 Workflow

檔案位置：`.github/workflows/deploy.yml`

觸發條件：
- Push 到 `main` 分支自動佈署
- 支援手動觸發（GitHub Actions 頁面 → Run workflow）

流程：CI Checks（typecheck + lint + test）→ 通過後自動 `wrangler deploy`

### GitHub Secrets 設定

在 repo 的 **Settings → Secrets and variables → Actions** 加入：

| Secret | 說明 | 取得方式 |
|--------|------|---------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API Token | [Dashboard → My Profile → API Tokens](https://dash.cloudflare.com/profile/api-tokens)，使用 **Edit Cloudflare Workers** 模板 |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare Account ID | Dashboard 右側欄或 `wrangler whoami` |

### 手動觸發佈署

GitHub repo → **Actions** → 左側 **Deploy to Cloudflare** → 右側 **Run workflow** → **Run workflow**

## 5. Cloudflare Secrets 設定

所有 secrets 透過 `wrangler secret put <NAME>` 設定。設定 secret 後 Cloudflare 會自動建立新版本，但 container 需要重新佈署才會生效。

### 設定順序（建議按此順序操作）

#### Step 1：AI Provider

```bash
wrangler secret put GOOGLE_AI_STUDIO_API_KEY
# 貼上 Google AI Studio API Key
```

#### Step 2：Gateway 認證

```bash
wrangler secret put MOLTBOT_GATEWAY_TOKEN
# 貼上隨機產生的 token
```

產生隨機 token：
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

> **重要**：記下這個 token，首次配對時需要使用。

#### Step 3：Cloudflare Access

```bash
wrangler secret put CF_ACCESS_TEAM_DOMAIN
# 格式：<你的team名>.cloudflareaccess.com

wrangler secret put CF_ACCESS_AUD
# 從 CF Access Application 設定頁取得（見第 6 節）
```

#### Step 4：R2 持久化

```bash
wrangler secret put R2_ACCESS_KEY_ID
# 貼上 R2 API Token 的 Access Key ID

wrangler secret put R2_SECRET_ACCESS_KEY
# 貼上 R2 API Token 的 Secret Access Key

wrangler secret put CF_ACCOUNT_ID
# 貼上你的 Cloudflare Account ID（wrangler whoami 可查看）
```

#### Step 5：Debug Routes（選用）

```bash
wrangler secret put DEBUG_ROUTES
# 輸入 true
```

#### Step 6：Telegram（選用）

```bash
wrangler secret put TELEGRAM_BOT_TOKEN
# 貼上 Telegram Bot Token

wrangler secret put TELEGRAM_DM_POLICY
# 輸入 pairing
```

### 完整 Secrets 總覽

| Secret | 用途 | 必要? |
|--------|------|-------|
| `GOOGLE_AI_STUDIO_API_KEY` | Google Gemini AI 模型 | 至少一個 AI Provider |
| `ANTHROPIC_API_KEY` | Anthropic Claude | 替代方案 |
| `OPENAI_API_KEY` | OpenAI GPT | 替代方案 |
| `MOLTBOT_GATEWAY_TOKEN` | Gateway 存取認證 | **是** |
| `CF_ACCESS_TEAM_DOMAIN` | Cloudflare Access 網域 | **是**（生產環境） |
| `CF_ACCESS_AUD` | Cloudflare Access AUD | **是**（生產環境） |
| `R2_ACCESS_KEY_ID` | R2 存取金鑰 | 強烈建議 |
| `R2_SECRET_ACCESS_KEY` | R2 秘密金鑰 | 強烈建議 |
| `CF_ACCOUNT_ID` | Cloudflare Account ID | R2 需要 |
| `DEBUG_ROUTES` | 啟用 /debug/* 路由 | 選用（除錯用） |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token | 選用 |
| `TELEGRAM_DM_POLICY` | Telegram DM 存取政策 | 選用（預設 pairing） |

## 6. Cloudflare Access 設定

### 建立 Access Application

1. 到 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com)
2. **Access → Applications → Add an application**
3. 選 **Self-hosted**
4. 設定 domain：`moltbot-sandbox.<subdomain>.workers.dev`
5. 建立 Policy（例如只允許特定 email）
6. 建立後複製 **Application Audience (AUD) Tag**

### 設定 Secrets

```bash
wrangler secret put CF_ACCESS_TEAM_DOMAIN
# 輸入：<team>.cloudflareaccess.com

wrangler secret put CF_ACCESS_AUD
# 輸入：Application Audience (AUD) Tag
```

## 7. R2 持久化儲存設定

R2 用於在重新佈署後保留 bot 的記憶、對話紀錄和設定。每 5 分鐘自動同步。

### 建立 R2 API Token

1. Cloudflare Dashboard → **R2 Object Storage → Overview**
2. 右下角 **Account Details → API Tokens → Manage**
3. **Create User API Token**
4. 設定：
   - **Permissions**: **Object Read & Write**（必須可寫入，選 Read only 會導致同步失敗）
   - **Specify bucket(s)**: **Apply to specific buckets only** → 選 `moltbot-data`
   - **TTL**: Forever
5. 建立後複製 **Access Key ID** 和 **Secret Access Key**

> **注意**：建立後只會顯示一次 Secret Access Key，請務必立即複製保存。

### 清理舊 Token

如果有之前建立的多餘 R2 API Token，建議刪除保持乾淨。上方的 **Account API Token**（如 Cloudflare Workers）不要動，那是給佈署用的。

### 設定 Secrets

```bash
wrangler secret put R2_ACCESS_KEY_ID
# 貼上 Access Key ID

wrangler secret put R2_SECRET_ACCESS_KEY
# 貼上 Secret Access Key

wrangler secret put CF_ACCOUNT_ID
# 貼上你的 Account ID（wrangler whoami 可查看）
```

### 同步機制

- Container 內的 `/root/.openclaw/`（config）、`/root/clawd/`（workspace）、`/root/clawd/skills/`（skills）
- 透過 rsync 同步到 R2 的 `/data/moltbot/` 掛載點
- 每 5 分鐘自動觸發（wrangler.jsonc 中的 cron trigger）
- 重啟時自動從 R2 恢復

## 8. Telegram 設定

### 建立 Telegram Bot

1. 在 Telegram 找 **@BotFather** 對話
2. 發送 `/newbot`
3. 設定 bot 名稱和 username
4. 複製拿到的 **Bot Token**

### 設定 Secrets

```bash
wrangler secret put TELEGRAM_BOT_TOKEN
# 貼上 Bot Token

wrangler secret put TELEGRAM_DM_POLICY
# 輸入 pairing（推薦）
```

### DM Policy 選項

| Policy | 說明 |
|--------|------|
| `pairing`（推薦） | 新使用者需要配對碼，經批准後才能使用 |
| `allowlist` | 只有白名單內的人可以使用 |
| `open` | 任何人都能使用（不建議） |
| `disabled` | 完全關閉 DM |

### 設定後需要重新佈署

Telegram 設定需要重新佈署才會生效：
- Push 到 main 觸發自動佈署，或
- GitHub Actions → Run workflow 手動觸發

## 9. 首次啟動與配對

### 佈署

```bash
git push origin main
# GitHub Actions 自動佈署
```

或手動觸發：GitHub repo → Actions → Deploy to Cloudflare → Run workflow

### Container 冷啟動

首次或重新佈署後需要冷啟動，約需 1-3 分鐘。頁面會顯示 loading 畫面，耐心等待。

### Web Gateway 配對

瀏覽器打開（帶上你的 Gateway Token）：
```
https://moltbot-sandbox.<subdomain>.workers.dev?token=<你的MOLTBOT_GATEWAY_TOKEN>
```

首次配對後之後不用再帶 token（CF Access 會自動注入）。

### Telegram 配對

1. 在 Telegram 對你的 bot 發送任意訊息
2. Bot 會回覆配對碼，例如：`Pairing code: KDSU8RWQ`
3. 批准配對（兩種方式擇一）：

**方式 A：透過 Admin UI**
```
https://moltbot-sandbox.<subdomain>.workers.dev/_admin/
```
找到 Devices 頁面，點 Approve。

**方式 B：透過 Debug CLI（如果 Admin UI 找不到待配對裝置）**
```
https://moltbot-sandbox.<subdomain>.workers.dev/debug/cli?cmd=openclaw%20pairing%20approve%20telegram%20<配對碼>
```
例如：
```
/debug/cli?cmd=openclaw%20pairing%20approve%20telegram%20KDSU8RWQ
```

> **注意**：Debug CLI 方式需要先啟用 `DEBUG_ROUTES=true`。

4. 看到 `Approved telegram sender <user_id>` 即表示成功
5. 回 Telegram 就能正常對話了

## 10. 安全性設定

### 防護層架構

```
使用者 → Cloudflare Edge → CF Access JWT 驗證 → Worker → Gateway Token → Sandbox Container → OpenClaw
```

| 防護層 | 作用 |
|--------|------|
| Cloudflare Access | 只有通過身份驗證的人能存取 Web Gateway |
| Gateway Token | 二次認證，防止未授權 WebSocket 連線 |
| Container 沙箱 | OpenClaw 跑在隔離的 micro-VM 裡 |
| 裝置配對 | 新裝置/Telegram 用戶需要配對才能互動 |
| Google 免費額度上限 | 自動限制 API 用量，防止濫用 |
| Telegram DM Policy | pairing 模式確保只有核准的人能用 |

### 安全建議

1. **關閉 DEBUG_ROUTES**：確認一切正常後 `wrangler secret delete DEBUG_ROUTES`
2. **不要設定 DEV_MODE=true**：會跳過所有認證
3. **定期輪換 Token**：`wrangler secret put MOLTBOT_GATEWAY_TOKEN` 更新
4. **Google AI Studio 用量上限**：免費額度預設限制即為保護
5. **Telegram DM Policy 用 pairing**：不要設成 open
6. **Cloudflare Rate Limiting**（選用）：Dashboard → Security → WAF → Rate limiting rules

### 事件回應 Checklist

如果懷疑被入侵：
1. 輪換 `MOLTBOT_GATEWAY_TOKEN`
2. 輪換 `GOOGLE_AI_STUDIO_API_KEY`
3. 輪換 R2 API Token（R2_ACCESS_KEY_ID + R2_SECRET_ACCESS_KEY）
4. 到 `/_admin/` 撤銷可疑的裝置配對
5. 設定 `TELEGRAM_DM_POLICY=disabled` 暫時關閉 Telegram
6. 重新佈署

## 11. Debug 與除錯

### 啟用 Debug Routes

```bash
wrangler secret put DEBUG_ROUTES
# 輸入 true
```

### 可用的 Debug 端點

| 端點 | 說明 |
|------|------|
| `/debug/processes?logs=true` | 查看所有 process 的 stdout/stderr（最重要！） |
| `/debug/logs` | 查看 OpenClaw Gateway 的 log |
| `/debug/logs?id=<processId>` | 查看特定 process 的 log |
| `/debug/env` | 查看環境變數（已脫敏） |
| `/debug/container-config` | 查看容器內的 openclaw.json |
| `/debug/version` | 查看 OpenClaw 和 Node 版本 |
| `/debug/gateway-api?path=/` | 探測 Gateway HTTP API |
| `/debug/cli?cmd=<指令>` | 在容器內執行 CLI 指令（需 URL encode） |
| `/debug/ws-test` | WebSocket 互動除錯頁面 |

### 常用除錯流程

1. **啟動失敗** → `/debug/processes?logs=true` 看 stderr
2. **Config 問題** → `/debug/container-config` 看實際設定
3. **環境變數問題** → `/debug/env` 確認 key 有傳入
4. **Gateway 無回應** → `/debug/gateway-api?path=/` 探測
5. **Telegram 配對** → `/debug/cli?cmd=openclaw%20pairing%20approve%20telegram%20<CODE>`
6. **R2 掛載狀態** → `/debug/processes?logs=true` 看 stdout 中的 mount 訊息

## 12. 踩坑紀錄

### 坑 1：openclaw onboard 不支援 --google-ai-studio-api-key

**症狀**：`start-openclaw.sh` 啟動失敗，stderr 顯示 `error: unknown option '--google-ai-studio-api-key'`

**原因**：OpenClaw 的 onboard CLI 沒有 `--google-ai-studio-api-key` 選項

**解法**：使用原生支援的 `--auth-choice gemini-api-key --gemini-api-key` 旗標

### 坑 2：google-ai-studio vs google-generative-ai API 類型

**症狀**：用 `api: 'openai-completions'` 設定 Google AI Studio 可能有相容性問題

**原因**：OpenClaw 原生支援 Google Gemini，應使用 `api: 'google-generative-ai'`

**解法**：
- Provider 名稱用 `google-gemini`（非 `google-ai-studio`）
- API 類型用 `google-generative-ai`（非 `openai-completions`）
- Base URL 用 `https://generativelanguage.googleapis.com/v1beta`（不含 `/openai/`）

### 坑 3：Container application namespace 衝突

**症狀**：佈署時報錯 `There is already an application with the name moltbot-sandbox-sandbox deployed that is associated with a different durable object namespace`

**原因**：本地佈署建立的 container 綁定了舊的 DO namespace，CI 佈署產生了新的 namespace

**解法**：
```bash
# 列出 containers
npx wrangler containers list
# 刪除衝突的 container
npx wrangler containers delete <container-id>
# 重新佈署
```

### 坑 4：Worker 環境變數驗證未包含 GOOGLE_AI_STUDIO_API_KEY

**症狀**：設定了 `GOOGLE_AI_STUDIO_API_KEY` 但 Worker 仍顯示 Configuration Error 頁面

**原因**：`src/index.ts` 的 `validateRequiredEnv()` 沒有檢查 `GOOGLE_AI_STUDIO_API_KEY`

**解法**：在驗證邏輯中加入 `const hasGoogleAIStudioKey = !!env.GOOGLE_AI_STUDIO_API_KEY`

### 坑 5：R2 API Token 權限不足

**症狀**：R2 掛載成功但同步失敗

**原因**：建立 R2 API Token 時選了 **Object Read only**

**解法**：權限必須選 **Object Read & Write**

### 坑 6：agents.defaults.tools 不被 OpenClaw 2026.2.3 認可

**症狀**：Gateway 啟動失敗，stderr 顯示 `agents.defaults: Unrecognized key: "tools"`

**原因**：OpenClaw 2026.2.3 的 config schema 不支援 `agents.defaults.tools`，嚴格驗證會拒絕未知的 key

**解法**：移除 `agents.defaults.tools` 相關設定。不要在 `openclaw.json` 中加入 OpenClaw 不認識的 key。

### 坑 7：DEBUG_ROUTES 需要單獨設定

**症狀**：`/debug/*` 路由回傳 `{"error":"Debug routes are disabled"}`

**原因**：DEBUG_ROUTES 預設未啟用

**解法**：`wrangler secret put DEBUG_ROUTES` 輸入 `true`

### 坑 8：Telegram 配對在 /_admin/ 頁面看不到

**症狀**：Telegram 已發送配對碼，但 `/_admin/` 頁面重新整理多次都看不到待配對裝置

**原因**：Admin UI 的裝置列表可能有延遲或 UI 問題

**解法**：使用 Debug CLI 直接批准配對：
```
/debug/cli?cmd=openclaw%20pairing%20approve%20telegram%20<配對碼>
```

### 坑 9：R2 同步失敗「Sync aborted: no config file found」

**症狀**：Admin UI 按 Backup Now 出現紅色錯誤「Sync aborted: no config file found」，但 debug CLI 確認 `/root/.openclaw/openclaw.json` 確實存在

**原因**：`syncToR2` 函式用 `test -f` 指令後檢查 `process.exitCode`，但 Cloudflare Sandbox 的 process API 對快速完成的指令可能不會正確設定 `exitCode`（為 `null` 或 `undefined`），導致 `exitCode !== 0` 永遠為 `true`

**解法**：改用 `test -f ... && echo FOUND` 並檢查 stdout 是否包含 `FOUND`，不依賴不可靠的 `exitCode`（已在 `src/gateway/sync.ts` 修正）

### 坑 10：Google AI Studio 免費額度比預期少

**症狀**：RPM=5、RPD=20，比官方文件標示的 RPM=10、RPD=500 少很多

**原因**：新帳號或特定地區的免費額度可能較低

**解法**：到 Google AI Studio 查看實際額度，必要時申請提升或啟用付費方案

---

## 快速部署 Checklist

```
[ ] Fork repo 到自己的 GitHub 帳號
[ ] Clone 並確認 npm install / npm test 通過
[ ] 取得 Google AI Studio API Key
[ ] 設定 GitHub Secrets：
    [ ] CLOUDFLARE_API_TOKEN
    [ ] CLOUDFLARE_ACCOUNT_ID
[ ] 建立 Cloudflare Access Application，取得 AUD Tag
[ ] 建立 R2 API Token（Object Read & Write，指定 moltbot-data bucket）
[ ] 產生隨機 MOLTBOT_GATEWAY_TOKEN 並記下
[ ] 設定 Wrangler Secrets（按順序）：
    [ ] GOOGLE_AI_STUDIO_API_KEY
    [ ] MOLTBOT_GATEWAY_TOKEN
    [ ] CF_ACCESS_TEAM_DOMAIN
    [ ] CF_ACCESS_AUD
    [ ] R2_ACCESS_KEY_ID
    [ ] R2_SECRET_ACCESS_KEY
    [ ] CF_ACCOUNT_ID
    [ ] DEBUG_ROUTES（設 true，除錯完再刪）
    [ ] TELEGRAM_BOT_TOKEN（選用）
    [ ] TELEGRAM_DM_POLICY（選用，設 pairing）
[ ] Push 到 main 觸發自動佈署
[ ] 等待冷啟動完成（1-3 分鐘）
[ ] 用 /debug/processes?logs=true 確認啟動成功
[ ] Web Gateway 首次存取帶 token 完成配對
[ ] Telegram 發訊息取得配對碼，用 debug CLI 批准
[ ] 確認 bot 正常回應
[ ] Admin UI 按 Backup Now 確認 R2 備份成功（或等 5 分鐘 cron 自動同步）
[ ] 用 /debug/cli?cmd=cat%20/data/moltbot/.last-sync 確認有時間戳
[ ] 關閉 DEBUG_ROUTES：wrangler secret delete DEBUG_ROUTES
```
