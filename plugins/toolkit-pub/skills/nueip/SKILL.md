---
name: nueip
version: 2.0.0
description: |
  NUEiP 一鍵查詢：今日打卡 / 本週 / 過去 30 天 / 部門請假 / 我的待簽 / 假期餘額 / 三合一晨報。
  單一入口 `/toolkit-pub:nueip <sub>`，省去記 MCP 工具名。

  **Triggers:** nueip, NUEiP, 打卡, 出勤, 請假, 待簽, 簽核, 假期餘額, 我的出勤, 部門請假
argument-hint: "today | week | recent | balance | me | team | pending | brief（不帶: 顯示選單）"
user-invocable: true
---

# NUEiP — 一鍵查詢

底層走 NUEiP 官方 MCP（`https://mcp.nueip.com/mcp`），工具名前綴 `mcp__claude_ai_NUEIP__`。
連線需 `mcp:use`、唯讀需 `mcp:read` 授權。

---

## 子指令對照表

| Primary | 中文 / 英文別名 | 用途 | 官方 MCP 工具 |
|---------|----------------|------|--------------|
| `today` | 今天、今日、打卡 | 今天打卡狀況 | `get_attendance_records` |
| `week` | 本週、這週 | 本週一到今天 | `get_attendance_records` |
| `recent` | 近一個月、近30天、一個月、last30、month、月、30天、過去30天 | 過去 30 天 + 異常標記 | `get_attendance_records` + `list_my_leaves` |
| `balance` | 假期餘額、餘額、特休、假期 | 我的假期餘額 | `get_leave_balance` |
| `me` | 我是誰、帳號、我的帳號、whoami | 帳號 / 部門 / 員編 | `get_attendance_records` + `get_employee` |
| `team` | 部門請假、誰請假、誰沒來、who-out、leaves、請假、部門 | 部門今日請假（🔒 主管） | `list_subordinate_leaves_today` |
| `team-attend` | 部門出勤、成員出勤、出勤記錄、出勤稽核、本月出勤、monthly-attend | 部門一段期間出勤稽核，套用彈性工時算法（🔒 主管） | `get_attendance_records`(dept) |
| `pending` | 待簽、待簽核、待我簽、to-sign、approvals、簽核 | 我待簽核的所有單（🔒 主管） | `list_pending_approvals` |
| `brief` | 晨報、早報、早會、morning | 今日打卡 + 待簽 + 今日請假 + 本月異常（部分 🔒）| 多項合一 |

argument 大小寫不分，中英別名都支援，舊命名（`month` / `leaves` / `approvals` / `morning` / `whoami`）保留為別名。比對不上就走「無 sub-command」流程。

🔒 **主管限定的子指令**（`team`、`team-attend`、`pending`，連帶影響 `brief` 中對應段落）打的是 manager 專屬範圍，非主管帳號會拿到空陣列或權限錯誤。

---

## 共通：日期計算（macOS）

- **今日**：從系統 context 的 `currentDate` 直接取，格式 `YYYY-MM-DD`
- **本週一**：`date -v-Mon +%Y-%m-%d`（若今天就是週一，這指令會回今天）
- **過去 30 天起**：`date -v-30d +%Y-%m-%d`
- 不要硬寫日期，每次跑都重算

官方多數工具的日期參數**必填或不吃「今天」語意**——後端沒有公司時區的 today 模式，一律明確帶日期，不要依賴預設。

---

## 共通：查詢範圍上限

`get_attendance_records` 與 `list_my_leaves` 的**跨度上限都是 92 天**。超過就依月份切段分批查，不要把使用者要的期間截短。

`get_attendance_records` 的分頁是**對使用者清單**分頁，不是對日期——查本人時 `total` 會是 1，不代表只有一天資料。

---

## 共通：自己的 user_id 怎麼來

官方沒有 whoami。要拿自己的 hashid，用不帶 `user_ids` 的查詢，回傳的就是本人：

```
mcp__claude_ai_NUEIP__get_attendance_records(date_from=<今日>, date_to=<今日>)
→ items[0].user_sn 即本人 hashid，items[0].user_no 是員編，items[0].dept_name 是部門名
```

若當日完全無排班資料導致 `items` 為空，改用 `mcp__claude_ai_NUEIP__get_leave_balance()`，回傳的 `balances[0].user_id` 同樣是本人。

---

## 共通：彈性工時判定

官方出勤只回**原始打卡與排班時段**，不回算好的遲到 / 早退。判定一律自行計算：

```
FLEX_LATE_THRESHOLD = 10:00   # 上班打卡超過此時才算遲到
REQUIRED_WORK_MIN   = 480     # 每日須滿工時（分鐘）

on  = punch_times 中 type=="on"  的最早一筆
off = punch_times 中 type=="off" 的最晚一筆

午休分鐘 = shift_relax_times 與 [on, off] 的交集長度
工作分鐘 = (off − on) − 午休分鐘

遲到分鐘 = max(0, minutes(on) − minutes(FLEX_LATE_THRESHOLD))
早退分鐘 = max(0, REQUIRED_WORK_MIN − (工作分鐘 + 當日請假分鐘))
```

- **午休取交集而非固定扣 60**：`shift_relax_times` 給的是當日實際午休區間；下午請假早走的日子午休未必被完整涵蓋，固定扣會低估工作分鐘、誤判早退。
- **回傳裡的 `punch_total_time` 不要拿來判定**。它與上式偶有數分鐘落差（扣抵規則未公開），本 skill 以自算值為準；與 NUEiP 頁面差幾分鐘屬正常。
- **工作分鐘不含請假時數**，早退判定必須把當日請假分鐘加回來，否則每個請假日都會被誤判成早退。

### 每日分類

| 條件 | 分類 |
|------|------|
| `shift_times` 為空陣列 | 無排班（週末 / 國定假日），**跳過不列** |
| `punch_times` 為空 且 當日請假 ≥ `shift_total_time` | 請假（全天） |
| `punch_times` 為空 且 無足額假單 | 缺卡 ⚠️ |
| 有打卡 且 有假單 | 半天假＋出勤（仍套用工時公式） |
| 有打卡 且 無假單 | 正常出勤（計算遲到 / 早退分鐘） |

`shift_times` 空陣列直接反映排班，**不必另外查行事曆或自建假日表**。

---

## 各子指令執行細節

### `today`（今天、打卡）

```
mcp__claude_ai_NUEIP__get_attendance_records(date_from=<今日>, date_to=<今日>)
```

輸出表格：排班 / 上班 / 下班 / 工時 / 狀態。

- `punch_times` 為空 → 顯示「今天還沒打卡」
- 判定出遲到 / 早退 / 缺卡 → 用 ⚠️ 標出來，並把分鐘數寫清楚

### `week`（本週）

```
start = $(date -v-Mon +%Y-%m-%d)
end   = <今日>
```

**平行**呼叫兩支：

```
mcp__claude_ai_NUEIP__get_attendance_records(date_from=start, date_to=end)
mcp__claude_ai_NUEIP__list_my_leaves(date_from=start, date_to=end)
```

輸出：
- 每日一列表格（同 `today` 欄位）
- 表尾：本週累計工時、異常天數小計

### `recent`（近一個月、過去 30 天）

```
start = $(date -v-30d +%Y-%m-%d)
end   = <今日>
```

**平行**呼叫 `get_attendance_records` 與 `list_my_leaves`（參數同上）。

> `list_my_leaves` 吃**區間**，查本人請假一次就夠——不要為了本人資料去逐日呼叫 `list_subordinate_leaves_today`。

輸出：
- 每日一列表格（無排班日標出但只顯示「無排班」）
- 表尾「異常摘要」：缺卡 N 天、早退 N 天、遲到 N 天、請假 N.N 天（依假別分列）
- 對**沒有對應假單的缺卡 / 早退 / 遲到**特別列出，提示「待補假單」
- 出勤紀律百分比：(正常天數 / 有排班天數) × 100%

### `balance`（假期餘額、特休）

```
mcp__claude_ai_NUEIP__get_leave_balance()
```

回傳 `balances[0].vacation_resource[]`，**每個授予期間各一筆**——同一假別會有多筆（例如加班換休按每次加班分別授予，可達數十筆）。

**必須自行聚合**，官方不做 UI 的合併：

1. 依 `vacation_name` group by
2. 每組加總 `remain_time` / `used_time` / `available_time`（單位皆為**分鐘**，480 = 1 天）
3. 期間顯示為該組的 `min(period_start_date)` ~ `max(period_end_date)`

**髒資料防護**（實測存在）：

- 有的紀錄 `period_start_date` 是 `"簽核中"` 這類非日期字串，`period_end_date` 為空字串
- 數值欄位可能是字串 `"-"` 而非數字

聚合前一律做數值強制轉換，非數字視為 0；期間非合法日期者不納入 min/max 計算，也不要顯示成日期。

輸出表格：假別 / 剩餘 / 已用 / 配額 / 期間。剩餘為負數時（換休超用）照實顯示，不要夾成 0。

### `me`（我是誰、帳號）

**平行**呼叫：

```
mcp__claude_ai_NUEIP__get_attendance_records(date_from=<今日>, date_to=<今日>)
mcp__claude_ai_NUEIP__get_employee(user_ids=<本人 hashid>)
```

第二支需要先有 hashid；若尚未取得，先跑第一支拿 `user_sn` 再跑第二支。

輸出：姓名 / 員編（`user_no`）/ 部門（`dept_name`）/ 在職狀態（`work_status`）/ hashid。

### 共通：部門 hashid 怎麼來（🔒 主管子指令前置）

`team` 與 `team-attend` 需要 `dept_ids`。**不要寫死**，每次動態解析：

1. 由「自己的 user_id」流程取得 `dept_name`
2. `mcp__claude_ai_NUEIP__get_org_tree()` 取部門樹
3. 在樹中（含各層 `sub_department_list`）找 `name` 等於該 `dept_name` 的節點，取其 `department_id`

找不到對應節點就直接說明「無法定位所屬部門」，**不要猜一個 department_id 去查**。

### `team-attend`（部門出勤稽核）🔒 主管

```
mcp__claude_ai_NUEIP__get_attendance_records(
  dept_ids  = <部門 hashid>,
  date_from = <起日>,
  date_to   = <訖日>
)
```

**日期參數解析**（從 argument 取）：
- argument 含兩個 `YYYY-MM-DD` → 直接使用
- argument 含一個 `YYYY-MM-DD` → 當作 start_date，end_date = 今日
- 無日期 → start_date = 本月 01 日（`date -v1d +%Y-%m-%d`），end_date = 今日

接著查同期間的部門請假。官方**沒有「部門 × 區間」的請假查詢**，只能逐日；
**只對有排班的日子發**（上一步回傳中 `shift_times` 非空者），並行送出：

```
mcp__claude_ai_NUEIP__list_subordinate_leaves_today(
  date     = <該日>,
  dept_ids = <部門 hashid>,
  limit    = 50
)
```

依 `user_id` + `date` 聚合成「每人每日請假分鐘數」：同日多筆要 sum；`total_time` 是分鐘字串，轉數字再加；**只採計 `sign_status == "2"`（已通過）**，簽核中的假不抵銷早退。

`dropped[]` 若非空表示有對象超出可見範圍，要在報表註明而不是當作沒事。

請假紀錄只回 `user_id`，姓名靠 `mcp__claude_ai_NUEIP__list_department_members(dept_id=<部門 hashid>)` 建對照表還原，**不要顯示裸 hashid**。

#### 輸出格式

```
部門出勤稽核 YYYY-MM-DD ～ YYYY-MM-DD（有排班日 N 天）

摘要：
| 成員 | 到勤 | 請假 | 缺卡 | 遲到(次/合計分) | 早退(次/合計分) |
|------|------|------|------|-----------------|-----------------|
| <成員 A> | N | N | N | N / Nm | N / Nm |
...

缺卡明細（需確認補假單）：
<日期>：<成員> / <成員> / ...

遲到明細：
<日期> <成員>：上班 HH:MM（遲 Nm）

早退明細：
<日期> <成員>：下班 HH:MM（早 Nm，工時 Xh Ym）
```

異常為 0 時省略對應明細段落。

### `team`（部門請假、誰沒來）🔒 主管

```
mcp__claude_ai_NUEIP__list_subordinate_leaves_today(
  date     = <今日>,
  dept_ids = <部門 hashid>,
  limit    = 50
)
```

`date` **必填**，明確帶今日日期。工具名稱雖為 `subordinate`，實測**含主管本人**。

非主管帳號會拿到空陣列或權限錯誤。若 `items` 為空，先說「沒有人請假，或你不是主管」再讓使用者自己判斷，不要假設一定有人。

輸出：
- 表格：申請人 / 假別（`rule_name`）/ 起迄（`start_time`–`end_time`）/ 工時（`total_time` 分鐘換算）/ 備註（`remark`）
- `user_id` 一律用 `list_department_members` 還原成姓名再輸出

> **簽核中的假沒有等價欄位**：官方回傳無 badges 統計，也沒有狀態篩選開關。不要輸出「另有 N 件簽核中」這類數字，也不要自行估算——拿不到就不寫。

### `pending`（待簽、簽核）🔒 主管

```
mcp__claude_ai_NUEIP__list_pending_approvals(limit=50)
```

一次回全部類型，**不需要**逐 type 補撈。回應含 `has_more` 與 `next_cursor`，`has_more` 為 true 就帶 `cursor` 續抓到完為止，不要只報第一頁。

非主管帳號會拿到空清單——直接顯示「沒有待簽，或你不是主管」並結束。

每件輸出：申請人（`user_name`）/ 類型（`type`）/ 段別（`section_typ`）/ 時間（`work_time`）/ 申請日（`c_date`）/ 備註（`remark`）。

最後總結：依 `type` 自行 group by 統計，例 `共 N 件，類別分佈：attendance × 2、leave × 1`。`total` 欄位是後處理完成的表單數，可用來對帳。

### `brief`（晨報、早會、早報）

**平行呼叫**（同一個 tool-call block 發出去，省 latency）：

1. `get_attendance_records(date_from=<今日>, date_to=<今日>)` — 今日打卡
2. `get_attendance_records(date_from=<今日-30>, date_to=<今日>)` — 月度資料
3. `list_my_leaves(date_from=<今日-30>, date_to=<今日>)` — 月度請假（對應異常用）
4. `list_pending_approvals(limit=50)` — 待簽

`team`（部門今日請假）需要部門 hashid，得先有 #1 的 `dept_name`，因此**排在第二批**送出。

非主管帳號的 #4 與部門請假會空，**仍要保留段落但顯示「無 / 你不是主管」**，#1～#3 不受影響照常顯示。

輸出格式（給人看，不是給機器）：

```
📅 今日打卡 (YYYY-MM-DD 週X)
  排班 09:00-18:00 / 上班 HH:MM / 下班 — / 還沒下班

📋 我待簽核 (N 件)        ← 非主管會是 0 件，照樣印標題
  - attendance × 2 (<申請人 A> 補卡 05-07, 05-11)
  - leave × 1
  ...

🏖️ 部門今日請假 (M 人)    ← 非主管會是「無權限 / 0 人」
  - <成員 A>｜特休全天 (travel)
  ...

⚠️ 近一個月異常 (過去 30 天)
  - 缺卡 1 天 (05-15) → 待補單
  - 早退 5 天 (05-06, 05-08, 05-19, 05-20)，未對應假單
  - 遲到 1 天 (05-06)
  - 病假 1.5 天 (已通過)
  - 紀律: 16/22 有排班日正常 = 73%
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

本 Skill 的 sub-command 一律**明確帶日期參數**給 MCP。若使用者另有「不帶參數時該查多少天」的個人習慣設定，本 Skill 帶的參數**完全覆蓋**它，不要疊加。

只有使用者跳過此 Skill、直接呼叫底層工具且不帶日期時，那條個人預設才生效。
