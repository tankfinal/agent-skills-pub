# agent-skills-pub

可分享的 Claude Code skills 集合，以 `toolkit-pub` plugin 形式發佈。未來會
陸續收錄更多 skills；目前公開的第一個是 `nueip`（NUEiP 一鍵查出勤 / 請假
/ 待簽 / 假期餘額）。

**Current version:** `toolkit-pub@2.0.0`

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

NUEiP 一鍵查詢。底下走 **NUEiP 官方 MCP server**（`https://mcp.nueip.com/mcp`）；
這個 skill 本身只是 wrapper，把子指令路由到對應的官方工具並套用彈性工時算法。

### 子指令

| 指令 | 別名 | 用途 | 權限 |
|------|------|------|------|
| `today` | `今天`、`今日`、`打卡` | 今天打卡狀況 | 🟢 一般員工 |
| `week` | `本週`、`這週` | 本週工時 | 🟢 一般員工 |
| `recent` | `近一個月`、`近30天`、`一個月`、`last30` | 過去 30 天（含早退 / 遲到 / 曠職 / 病假 / WFH 標記） | 🟢 一般員工 |
| `balance` | `假期餘額`、`餘額`、`特休`、`假期` | 假期餘額 | 🟢 一般員工 |
| `me` | `我是誰`、`帳號`、`我的帳號`、`whoami` | NUEiP 帳號資訊（除錯用） | 🟢 一般員工 |
| `team` | `部門請假`、`誰請假`、`誰沒來`、`who-out` | 部門今日請假 | 🔒 主管 |
| `pending` | `待簽`、`待簽核`、`待我簽`、`to-sign` | 我的待簽核（一次回全類型，自動續頁） | 🔒 主管 |
| `brief` | `晨報`、`早報`、`早會`、`morning` | 三合一晨報（今日打卡 + 待簽 + 部門請假 + 本月異常） | 部分 🔒 |

別名隨便混搭，不用記英文。也可以直接講白話「我特休還剩多少」、「今天誰沒來」、
「我有什麼待簽」— Claude 會自己路由。

> 🔒 標記的子指令查的是 manager 範圍的資料。可見範圍由 NUEiP 依你的帳號權限
> 判定，非主管帳號會拿到空陣列或權限錯誤。`brief`（晨報）三合一裡的「待簽」
> 「部門請假」兩塊也會空，但「今日打卡」「本月異常」一樣會跑出來。

### 前置：兩件事都要做

`nueip` 要動起來，**Claude Code 的 plugin** 跟 **NUEiP 官方 MCP 連線**兩個都要有。
少一個 `/toolkit-pub:nueip` 就會壞掉：

| 缺哪個 | 症狀 |
|--------|------|
| 沒裝 plugin | Claude 說「沒有 `toolkit-pub:nueip` 這個 skill」 |
| 沒連 MCP | `/toolkit-pub:nueip` 跑得起來，但說「找不到 `mcp__claude_ai_NUEIP__*` 工具」 |

**Step A — 裝 plugin**（如果上面「Install the plugin」還沒做就回去做）

**Step B — 連 NUEiP 官方 MCP**

NUEiP 官方提供 remote MCP server，endpoint 是：

```
https://mcp.nueip.com/mcp
```

在 Claude 的 connector 設定中連上它，走 OAuth 授權（連線需 `mcp:use`、
唯讀查詢需 `mcp:read`）。**不需要**在本機跑任何 server，也不需要把帳號密碼
存進 Keychain——授權由 NUEiP 端管理，可見範圍與你在 NUEiP UI 上的權限一致。

> 這項功能需要**你的公司已在 NUEiP 啟用 MCP**。開通與授權流程請洽貴公司的
> NUEiP 管理員或 NUEiP 官方（產品說明：<https://www.nueip.com/mcp>）——
> 各公司的開通方式不同，本 repo 不提供繞過授權的方法。

連好之後**完全重啟 Claude Code**，用這個確認連線狀態：

```bash
claude mcp list | grep -i nueip
```

看到 `✔ Connected` 就代表通了。再驗證整條鏈：

```
/toolkit-pub:nueip me
```

回傳你的姓名 / 部門 / 員編就代表 skill 也接上了。

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
skill / 新版本）就會 email 你。

---

## Troubleshooting

### 通用

**Claude 說「沒有 toolkit-pub:<skill> 這個 skill」**

→ Plugin 沒裝或沒 reload。`/plugin` 看 installed plugins 有沒有
`toolkit-pub`：

- 沒有 → 回去走「Install the plugin」
- 有 → `/reload-plugins`

### `nueip` 專屬

**`/toolkit-pub:nueip` 開始跑後，Claude 說「找不到 mcp__claude_ai_NUEIP__* 工具」**

→ MCP 沒接上。逐項檢查：

1. `claude mcp list | grep -i nueip` 有沒有出現、是不是 `✔ Connected`
2. 沒出現 → 回去做 Step B，確認 connector 已授權
3. 出現但 `✘ Failed` → OAuth 可能過期，重新授權一次
4. **完全重啟 Claude Code**（連線是 startup 時建立的，新設定要重開才生效）

**工時 / 遲到早退跟 NUEiP 頁面對不上**

正常。NUEiP 頁面以固定排班（通常 09:00–18:00）為基準，本 skill 一律套用彈性
工時口徑自行計算，兩者本來就不同。官方 MCP 也只回原始打卡與排班時段，不回
算好的遲到 / 早退。回傳裡的 `punch_total_time` 與自算值偶有數分鐘落差，本
skill 以自算值為準。

**假期餘額的數字看起來被拆成很多筆**

官方回的是**每個授予期間各一筆**（例如加班換休按每次加班分別授予），skill 會
自行依假別聚合。若看到負數剩餘，那是換休超用，照實顯示不是 bug。

**權限 / 顯示問題（看不到部門其他人的請假之類）**

可見範圍由 NUEiP 依你的帳號權限判定，跟你在 NUEiP UI 上看到的一致。如果 UI
上你本來就看不到，這邊也不會多看到。

---

## ⚠️ 排程環境未驗證

官方 MCP 是 remote HTTP server，理論上不受本地 stdio MCP 的限制。但
**Claude.ai 的 scheduled routine / Run Now / RemoteTrigger 尚未實測**，
不保證可用。目前請當本地工具用；若你實測可行，歡迎開 issue 回報。

---

## License & 用途範圍

MIT-ish — 自由使用、修改。請勿用於商業 NUEiP 自動化、shared machine、或任何
NUEiP TOS 不允許的場景。授權與可見範圍由 NUEiP 官方 MCP 依你的帳號權限控管，
本 repo 不碰你的帳號密碼。
