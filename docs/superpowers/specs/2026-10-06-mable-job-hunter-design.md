# Mable 自動找工作 — 設計文件

## 目標
在 Mac mini 上 24/7 運行 n8n,自動把 Mable 上「符合我條件」的 support worker 工作推送到手機,**由本人決定是否申請**。範圍僅限 Mable;Hireup / Seek / Facebook / 小紅書 之後各自另開 plan,共用 `RawJob` 與評估引擎。

## 不做的事(刻意)
- 不自動申請/自動回覆客人(平台條款風險、申請品質、帳號被封風險)。
- 不用瀏覽器自動化爬 Mable(需登入、脆弱、可能違反 Mable 使用條款)。先用 Mable 寄到我 Gmail 的職位提醒郵件當資料來源。若郵件資訊不足,再另開 plan 評估替代方案。
- 不呼叫 Google Maps 算車程;用「郊區白名單」代替。

## 我的篩選條件(使用者原話整理)
1. **類型**:想做的類型(例:園藝、家庭清潔)才要;其他類型不要。
2. **地點**:住 Parkwood East(Gold Coast, QLD)。可接受:附近郊區、輕軌(G:link)沿線郊區、火車線沿線郊區。
3. **時長**:單次 ≥ 2 小時。
4. **行事曆**:不能與 Google 日曆(工作 + 讀書)衝突。
5. **週二、週四**上午在 Bilinga 上課;**下課後**在學校附近(Bilinga 一帶)的工作也可以接。

## 流程
```
Gmail Trigger(Mable 職位提醒)
  → POST /parse           解析郵件 → RawJob[]
  → Google Calendar 取未來 4 週事件
  → POST /process         去重 + 評估 → Decision[](match / maybe / reject)
  → Telegram bot 傳訊息給自己(每個 match / maybe 一則,含連結與原因)
  → POST /seen            傳送成功後才標記已讀
每日摘要:收到幾封、解析幾個、match 幾個(用來發現「靜默失效」)
```

## 判定規則
- 四條規則(類型、時長、地點、日曆)全部執行,不短路,方便日後檢視 reject 原因。
- 任一條 `fail` → `reject`;無 fail 但有 `unknown` → `maybe`;全 `pass` → `match`。
- 彈性時間(flexible)工作:日曆規則回 `unknown`,最後是 `maybe`。

## 技術
n8n 以 Docker Desktop + docker compose 在本機運行(本機為 Apple Silicon Mac,目前尚未安裝 Docker)。評估服務:TypeScript + Node ≥ 20、vitest、zod、luxon、cheerio、fastify、better-sqlite3;n8n 與評估服務用 docker compose 一起跑;時區固定 `Australia/Brisbane`。

## 待使用者確認(plan 內以預設值實作,全部可在 `config/preferences.json` 修改)
- Mable 是否真的會寄職位提醒郵件?內容是否含:標題、類型、郊區、時間?(Task 1 的 gate)
- 想做的類型完整清單、要排除的關鍵字(例如 sleepover、需要抬舉病人)。
- 週二/週四下課時間(預設 13:00)。
- 通勤緩衝(預設 45 分鐘)。
- 通知管道:已決定用 Telegram(Gmail 只用來讀取 Mable 提醒郵件,不寄信)。
