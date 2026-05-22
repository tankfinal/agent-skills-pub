# agent-skills-pub

可分享的 Claude Code skills 集合，以 `toolkit-pub` plugin 形式發佈。未來會
陸續收錄更多 skills；目前公開的第一個是 `nueip`（NUEiP 一鍵查出勤 / 請假
/ 待簽 / 假期餘額）。

**Current version:** `toolkit-pub@1.0.0`

---

## What's here

| Skill | 狀態 | Triggers | 用途 |
|-------|------|----------|------|
| `nueip` | ✅ 已上架 | nueip, 打卡, 出勤, 請假, 待簽, 假期餘額 | NUEiP 一鍵查詢 — 單一入口 `/toolkit-pub:nueip <sub>` 路由 8 個子指令 |

> 之後加進來的 skill 會直接掛在同一個 `toolkit-pub` plugin 底下，裝過一次就
> 自動拿到新 skill（記得 `/plugin marketplace update` + reload）。

---

## Install the plugin

不管你想用哪個 skill，先把 plugin 裝起來：

```
/plugin marketplace add tankfinal/agent-skills-pub
/plugin install toolkit-pub@tankfinal-agent-skills-pub
```

裝完 **完全 quit Claude Code**（cmd+Q，不是 `/clear`）後重開。

> 各 skill 可能還有自己的前置需求（例如 `nueip` 要先裝 MCP server）。看下面
> 各 skill 的小節。

---

## Skill: `nueip`

NUEiP 一鍵查詢。底下靠 [`tankfinal/nueip-mcp`](https://github.com/tankfinal/nueip-mcp)
Python MCP server 真的去打 NUEiP API；這個 skill 本身只是 wrapper。

### 子指令

| 指令 | 別名 | 用途 | 權限 |
|------|------|------|------|
| `today` | `今天`、`今日`、`打卡` | 今天打卡狀況 | 🟢 一般員工 |
| `week` | `本週`、`這週` | 本週工時 | 🟢 一般員工 |
| `recent` | `近一個月`、`近30天`、`一個月`、`last30` | 過去 30 天（含早退 / 遲到 / 曠職 / 病假 / WFH 標記） | 🟢 一般員工 |
| `balance` | `假期餘額`、`餘額`、`特休`、`假期` | 假期餘額 | 🟢 一般員工 |
| `me` | `我是誰`、`帳號`、`我的帳號`、`whoami` | NUEiP 帳號資訊（除錯用） | 🟢 一般員工 |
| `team` | `部門請假`、`誰請假`、`誰沒來`、`who-out` | 部門今日請假 | 🔒 主管 |
| `pending` | `待簽`、`待簽核`、`待我簽`、`to-sign` | 我的待簽核（自動逐 type 補撈） | 🔒 主管 |
| `brief` | `晨報`、`早報`、`早會`、`morning` | 三合一晨報（今日打卡 + 待簽 + 部門請假 + 本月異常） | 部分 🔒 |

別名隨便混搭，不用記英文。也可以直接講白話「我特休還剩多少」、「今天誰沒來」、
「我有什麼待簽」— Claude 會自己路由。

> 🔒 標記的子指令底層打 NUEiP 的 manager / leader 專屬 endpoint
> （`personal_leave_application_manager`、`leader_audit_work_list`）。非主管
> 帳號呼叫會被 NUEiP 擋回，看到 401/403 或空陣列。`brief`（晨報）三合一裡
> 的「待簽」「部門請假」兩塊也會空，但「今日打卡」「本月異常」一樣會跑出來。

### 前置：兩件事都要做

`nueip` 要動起來，**Claude Code 的 plugin** 跟 **底層 NUEiP MCP server** 兩
個都要裝。少一個 `/toolkit-pub:nueip` 就會壞掉：

| 缺哪個 | 症狀 |
|--------|------|
| 沒裝 plugin | Claude 說「沒有 `toolkit-pub:nueip` 這個 skill」 |
| 沒裝 MCP server | `/toolkit-pub:nueip` 跑得起來，但說「找不到 `mcp__nueip__*` 工具」 |

**Step A — 裝 plugin**（如果上面「Install the plugin」還沒做就回去做）

**Step B — 裝 NUEiP MCP server**（一次就好，macOS）：

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/tankfinal/nueip-mcp/main/install.sh)
```

> 這條腳本**只裝 MCP server，不會幫你做 Step A 的 plugin install**。

腳本會：clone repo → uv sync → Keychain 存密碼（互動輸入）→ 問員編 / 公司簡碼 →
`claude mcp add` 註冊。公司簡碼就是你 NUEiP 登入頁 URL 的子網域（例：
`https://portal.nueip.com/login?ondemand=acme` → 簡碼 `acme`）。

> 不放心 `curl | bash`？看一遍腳本：[install.sh](https://github.com/tankfinal/nueip-mcp/blob/main/install.sh)。
> 或走 [nueip-mcp 的手動安裝](https://github.com/tankfinal/nueip-mcp#手動安裝)。

裝完一樣要 **完全重啟 Claude Code**，然後驗證：

```
/toolkit-pub:nueip me
```

回傳你的姓名 / 部門 / 員編就代表整條鏈都通了。

### 使用範例

```
/toolkit-pub:nueip brief      ← 早上開機跑這個（三合一晨報）
/toolkit-pub:nueip recent     ← 過去 30 天異常彙整（補假單用）
/toolkit-pub:nueip pending    ← 看待簽核
/toolkit-pub:nueip balance    ← 看特休還剩多少
```

或直接講白話：

```
今天誰沒來
我特休還剩多少
我有什麼待簽核
```

Claude 會 fuzzy match 到對應子指令。

---

## Update

```
/plugin marketplace update tankfinal-agent-skills-pub
/plugin update toolkit-pub@tankfinal-agent-skills-pub
```

> 實測：inline 的 `/plugin update ...` 偶爾沒回饋（silent no-op）。最保險是
> 先跑 `marketplace update`，再開 `/plugin` 互動式選單 → Installed → 選
> plugin → Update。

### 想第一時間收到新版通知

GitHub 右上角 **Watch → Custom → Releases**，這個 repo 出新 release（含新
skill / 新版本）就會 email 你。如果你有用 `nueip`，順手把
[`nueip-mcp`](https://github.com/tankfinal/nueip-mcp) 也 Watch 起來（不過它
的 `launch.sh` 已經內建每日自動檢查 + stderr 提醒）。

---

## Troubleshooting

### 通用

**Claude 說「沒有 toolkit-pub:<skill> 這個 skill」**

→ Plugin 沒裝或沒 reload。`/plugin` 看 installed plugins 有沒有
`toolkit-pub`：

- 沒有 → 回去走「Install the plugin」
- 有 → `/reload-plugins`

### `nueip` 專屬

**`/toolkit-pub:nueip` 開始跑後，Claude 說「找不到 mcp__nueip__* 工具」**

→ MCP server 沒接上。逐項檢查：

1. `~/.claude.json` 的 `mcpServers.nueip` 設定有照 nueip-mcp README
2. `launch.sh` 是 executable：`ls -la ~/nueip-mcp/launch.sh` 要有 `x`
3. Keychain 有存：`security find-generic-password -s nueip -a "$USER"` 不報錯就 OK
4. **完全重啟 Claude Code**（MCP 是 startup 時 spawn 的，新設定要重開才生效）

**結果跑出來但數字怪 / 假別餘額對不上 NUEiP UI**

通常是 NUEiP 改 schema。先到 [nueip-mcp issues](https://github.com/tankfinal/nueip-mcp/issues)
看有沒有人遇到，沒有就開新 issue。

**權限 / 顯示問題（看不到部門其他人的請假之類）**

`team_leaves` 跟 `pending_approvals` 看到的範圍跟你在 NUEiP UI 上的權限
一致。如果 UI 上你本來就看不到，這邊也不會多看到。

---

## ⚠️ 只能在本地跑

`nueip` 這條鏈（以及任何依賴 MCP 的 skill）：Claude.ai 的 scheduled routine
/ Run Now / RemoteTrigger **都不行** — 上游 MCP 注入 bug 會 silent fail。請
當本地工具用。

---

## License & 用途範圍

MIT-ish — 自由使用、修改。請勿用於商業 NUEiP 自動化、shared machine、或任何
NUEiP TOS 不允許的場景。Credentials 自負，作者不負保管責任。
