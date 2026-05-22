---
name: nueip
version: 1.0.0
description: |
  NUEiP 一鍵查詢：今日打卡 / 本週 / 過去 30 天 / 部門請假 / 我的待簽 / 假期餘額 / 三合一晨報。
  單一入口 `/toolkit-pub:nueip <sub>`，省去記 MCP 工具名。

  **Triggers:** nueip, NUEiP, 打卡, 出勤, 請假, 待簽, 簽核, 假期餘額, 我的出勤, 部門請假
argument-hint: "today | week | recent | balance | me | team | pending | brief（不帶: 顯示選單）"
user-invocable: true
---

# NUEiP — 一鍵查詢

⚠️ **本地專用**：此 Skill 依賴 NUEiP MCP server，**只能在本地 session 跑**。claude.ai 排程 / Run Now / RemoteTrigger 因為上游 MCP 注入 bug 會 silent fail，請當本地工具用。

---

## 子指令對照表

| Primary | 中文 / 英文別名 | 用途 | MCP 工具 |
|---------|----------------|------|---------|
| `today` | 今天、今日、打卡 | 今天打卡狀況 | `my_attendance(today)` |
| `week` | 本週、這週 | 本週一到今天 | `my_attendance(Mon..today)` |
| `recent` | 近一個月、近30天、一個月、last30、month、月、30天、過去30天 | 過去 30 天 + 異常標記 | `my_attendance(today-30..today)` |
| `balance` | 假期餘額、餘額、特休、假期 | 我的假期餘額 | `my_leave_balance` |
| `me` | 我是誰、帳號、我的帳號、whoami | NUEiP 帳號資訊 | `whoami` |
| `team` | 部門請假、誰請假、誰沒來、who-out、leaves、請假、部門 | 部門今日已通過請假（🔒 主管） | `team_leaves(today, dept, passed)` |
| `pending` | 待簽、待簽核、待我簽、to-sign、approvals、簽核 | 我待簽核的所有單（🔒 主管） | `pending_approvals(各 type)` |
| `brief` | 晨報、早報、早會、morning | 今日打卡 + 待簽 + 今日請假 + 本月異常（部分 🔒）| 多項合一 |

argument 大小寫不分，中英別名都支援，舊命名（`month` / `leaves` / `approvals` / `morning` / `whoami`）保留為別名。比對不上就走「無 sub-command」流程。

🔒 **主管限定的子指令**（`team`、`pending`，連帶影響 `brief` 中對應段落）底層打 NUEiP 的 manager / leader 專屬 endpoint，非主管帳號會被 NUEiP 擋回（401/403 或空陣列）。

---

## 共通：日期計算（macOS）

- **今日**：從系統 context 的 `currentDate` 直接取，格式 `YYYY-MM-DD`
- **本週一**：`date -v-Mon +%Y-%m-%d`（若今天就是週一，這指令會回今天）
- **過去 30 天起**：`date -v-30d +%Y-%m-%d`
- 不要硬寫日期，每次跑都重算

---

## 各子指令執行細節

### `today`（今天、打卡）

呼叫 `mcp__nueip__my_attendance()` 不帶任何參數。

輸出表格：排班 / 上班 / 下班 / 工時 / 狀態。

- `future:true` 且 `punch:[]` → 顯示「今天還沒打卡」
- 任一 tag（早退 / 遲到 / 缺卡 / 曠職）→ 用 ⚠️ 標出來，並把分鐘數寫清楚

### `week`（本週）

```
start = $(date -v-Mon +%Y-%m-%d)
end = <今日>
```

呼叫 `mcp__nueip__my_attendance(start_date=start, end_date=end)`。

輸出：
- 每日一列表格（同 `today` 欄位）
- 表尾：本週累計工時、異常天數小計

### `recent`（近一個月、過去 30 天）

```
start = $(date -v-30d +%Y-%m-%d)
end = <今日>
```

呼叫 `mcp__nueip__my_attendance(start_date=start, end_date=end)`。

輸出：
- 每日一列表格（工作日為主，週末/假日標出但只顯示 daytype）
- 表尾「異常摘要」：曠職 N 天、早退 N 天、遲到 N 天、病假 N.N 天、加班 N 分鐘、WFH N 次
- 對**沒有對應 timeoff 的早退/遲到/曠職**特別列出，提示「待補假單」
- 出勤紀律百分比：(正常天數 / 工作日數) × 100%

### `balance`（假期餘額、特休）

呼叫 `mcp__nueip__my_leave_balance()`（預設瘦身模式，數字已對齊 NUEiP UI，**不要再加 raw=true**）。

輸出：
- 表格：假別 / 剩餘 / 已用 / 配額 / 截止日
- 跨年度假別（特休、Extra Leave）若 UI 合計含下年度，加註「（其中下年度 X）」
- NUEiP 的 `my_leave_balance` 預設瘦身模式已對齊 UI 顯示邏輯（同一 `v_name` group_by、本期 only），數字直接報出即可，不要自己對 raw payload 做二次計算

### `me`（我是誰、帳號）

呼叫 `mcp__nueip__whoami()`，輸出 u_sn、員編、姓名、部門、職稱。

### `team`（部門請假、誰沒來）🔒 主管

呼叫 `mcp__nueip__team_leaves()` 不帶參數（預設 dept + passed + today）。

非主管帳號 NUEiP 會擋回 401/403 或空陣列。若 result 是空且 badges 全 0，先說「沒有人請假，或你不是主管」再讓使用者自己判斷，不要假設一定有人。

輸出：
- 表格：申請人 / 部門 / 假別 / 起迄 / 工時 / 備註 / 代理人
- 表尾顯示 badges：`passed:X / ongoing:Y / failed:Z`
- 若 `ongoing > 0`，主動提醒「另有 Y 件簽核中，要看的話跑 `/toolkit-pub:nueip team ongoing`」（這個分支不會自動展開，避免太雜）

### `pending`（待簽、簽核）🔒 主管

非主管帳號會收到全 0 的 badges，要 graceful handle — 直接顯示「沒有待簽，或你不是主管」並結束，不要做第二階段補撈。

兩階段：
1. 呼叫 `mcp__nueip__pending_approvals()` 預設（type=leave）→ 讀回應的 `badges`
2. 對 `badges` 中**值 > 0** 的類別（attendance / overtime / cancel_leave / fee / preovertime / flextime / substitue / cancel_leave_substitute / eform / leave_entitlement）逐一補撈
3. `leave` 那一筆若已是 0 就不再補

每件輸出：
- 申請人 / 日期 / 段別 / 申請時間 / 實際打卡（in / out）/ 補卡後仍差秒數 / 備註

最後總結：`共 N 件，類別分佈：attendance × 2、leave × 0 ...`

### `brief`（晨報、早會、早報）

**平行呼叫**（同一個 tool-call block 發出去，省 latency）：
1. `mcp__nueip__my_attendance()` — 今日
2. `mcp__nueip__pending_approvals()` — 預設 leave，先拿 badges
3. `mcp__nueip__team_leaves()` — 預設 dept today passed
4. `mcp__nueip__my_attendance(start_date=<今日-30>, end_date=<今日>)` — 月度資料

拿到 #2 的 badges 後，**序列地**對 > 0 的類別補撈（這部分不能平行，得等 badges 回來）。

非主管帳號的 #2、#3 會空，**仍要保留段落但顯示「無 / 你不是主管」**，#1、#4 不受影響照常顯示。

輸出格式（給人看，不是給機器）：

```
📅 今日打卡 (YYYY-MM-DD 週X)
  排班 09:00-18:00 / 上班 HH:MM / 下班 — / 還沒下班

📋 我待簽核 (N 件)        ← 非主管會是 0 件，照樣印標題
  - attendance × 2 (<申請人 A> 補卡 05-07, 05-11)
  - leave × 0
  - overtime × 0
  ...

🏖️ 部門今日請假 (M 人)    ← 非主管會是「無權限 / 0 人」
  - <成員 A>｜特休全天 (travel)
  ...

⚠️ 近一個月異常 (過去 30 天)
  - 曠職 1 天 (05-15) → 待補單
  - 早退 5 天 (05-06, 05-08, 05-19, 05-20)，未對應假單
  - 遲到 1 天 (05-06)
  - 病假 1.5 天 (已通過)
  - 紀律: 16/22 工作日正常 = 73%
```

不要每段都裝飾，保留 emoji 當 section anchor、表格簡潔。

---

## 沒有 sub-command 時

使用者輸入 `nueip` / `/toolkit-pub:nueip` 沒帶子指令、或 sub 無法對應到上表時，**不要亂猜跑哪個**，輸出選單請對方選：

```
NUEiP 子指令：
  today    / 今天          — 今天的打卡
  week     / 本週          — 本週工時
  recent   / 近一個月       — 過去 30 天（含異常）
  balance  / 假期餘額       — 假期餘額
  me       / 我的帳號       — NUEiP 帳號資訊
  team     / 部門請假       — 部門今日請假         🔒 主管
  pending  / 待簽          — 我的待簽核            🔒 主管
  brief    / 晨報          — 三合一早報            部分 🔒

範例: /toolkit-pub:nueip brief
```

---

## 預設值覆蓋規則

若使用者另有「`my_attendance` 不帶參數時應該預設查多少天」的習慣設定（例：個人 memory 寫了「預設 30 天」），本 Skill 的 sub-command **明確帶日期參數時**直接傳給 MCP，**完全覆蓋**該預設，不要疊加。

只有使用者跳過此 Skill、直接呼叫底層 `mcp__nueip__my_attendance` 不帶日期時，那條個人預設才生效。
