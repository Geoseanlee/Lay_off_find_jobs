# Mable Job Hunter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 Mable 職位提醒郵件自動解析、依我的偏好評估,並把 match / maybe 的工作用 Telegram 推送給自己(手機即時通知),24/7 跑在 Mac mini 的 n8n 上。

**Architecture:** n8n 只負責「接線」(Gmail 觸發、Google Calendar、Telegram 推送、排程)。Gmail 只當資料來源(唯讀),通知走 Telegram,兩條管道互相獨立:Google 憑證過期時,Telegram 仍能送出失效警報。所有判斷邏輯放在一個小型 TypeScript HTTP 服務(`/parse`、`/process`、`/seen`),這樣規則可以用 vitest 做 TDD,而不是藏在 n8n 的 Code node 裡。資料來源用 adapter 形式(`郵件 → RawJob[]`),日後加 Hireup / Seek 只需新增 adapter。

**Tech Stack:** Node ≥ 20(本機已有 24.19.0), TypeScript, vitest, zod, luxon, cheerio, fastify, better-sqlite3, Docker Desktop + docker compose, n8n(Docker image `n8nio/n8n`,穩定版)。

**Spec:** `docs/superpowers/specs/2026-10-06-mable-job-hunter-design.md`

## Global Constraints

- 時區固定 `Australia/Brisbane`(Gold Coast,無夏令時間);所有星期/時間判斷都用 `prefs.timezone`,不得依賴伺服器本地時區。
- 單次工作時長下限 `minDurationHours = 2`(剛好 2 小時算通過)。
- 日曆檢查範圍 `horizonWeeks = 4`;通勤緩衝 `travelBufferMinutes = 45`(兩端各加)。
- 週二、週四(`campus.days = [2, 4]`,ISO 星期一=1)下課後才可接 `campusArea` 郊區的工作;下課時間預設 `campus.classEndsAt = "13:00"`。
- 不自動申請、不爬登入後的頁面;只讀 Gmail 中 Mable 的提醒郵件。
- 工作 ID 去重:同一 `RawJob.id` 只推送一次;只有推送成功後才標記為已讀。
- 全部個人偏好放 `config/preferences.json`,程式碼中不得寫死郊區名單或類型名單。
- 通知管道:Telegram bot 傳訊息給自己。Bot token 只存在 n8n credentials(不進 git、不貼到聊天室);chat ID 非機密,直接填在 Telegram 節點參數(n8n 預設不允許節點讀環境變數,不要用 `.env`)。
- Gmail Trigger 的篩選必須是 `from:mable.com.au`;只用 Gmail 讀取權限,不要求寄信權限。
- n8n 只綁 `127.0.0.1:5678`,不對外網開放;Google OAuth 的 client secret 只存在 n8n credentials,不進 git。
- 本機環境(已確認):Apple Silicon (arm64)、macOS 27.0.1、Homebrew 7.0.7、Node 24.19.0、git 2.55.0;**尚未安裝 Docker**。

## Review Focus

1. **Mable 改了郵件版面 → 解析出 0 筆卻沒人知道**:預期是明確報錯並推送「解析失敗」警報,而不是靜默地當作「今天沒工作」。(Task 2、7、8)
2. **護理實習 / 考試整天事件**:日曆中名稱含 `placement`、`exam` 的全天事件要擋掉整天;生日、節日之類的全天事件不能擋。(Task 5)
3. **跨夜 / sleepover 工作(22:00–06:00)**:時長必須跨午夜計算為 8 小時,不能算成負數而被誤殺。(Task 3)
4. **郊區名稱寫法不一**(`SURFERS PARADISE`、`Surfers Paradise QLD 4217`、多餘空白):預期都能匹配到白名單。(Task 4)
5. **伺服器是 UTC 時區**:UTC 週一 23:00 在 Brisbane 已是週二 09:00,週二/週四的校園區規則和重複性工作的星期必須用 Brisbane 判斷。(Task 4、5)

另:同一份工作出現在多封郵件 → 只推送一次(Task 7)。

---

## File Structure

```
docker-compose.yml                 Task 0 建立(僅 n8n),Task 8 加入 job-filter
package.json / tsconfig.json / vitest.config.ts / .gitignore / .env.example
config/preferences.json            我的偏好(類型、郊區、時間、緩衝)
src/types.ts                       共用型別
src/config.ts                      loadPreferences:讀取 + zod 驗證 + 正規化
src/parse/mableEmail.ts            Mable 提醒郵件 HTML → RawJob[]
src/rules/category.ts              類型規則
src/rules/duration.ts              時長規則
src/rules/location.ts              地點規則(含週二/四校園區)
src/rules/calendar.ts              日曆規則(busy 區間、展開重複性工作、衝突判斷)
src/evaluate.ts                    組合四條規則 → Decision
src/store.ts                       SQLite:已讀去重 + 統計
src/server.ts                      Fastify:/health /parse /process /seen /stats
src/main.ts                        啟動入口
tests/**                           對應的 *.test.ts,tests/fixtures/ 放真實郵件樣本
Dockerfile                         評估服務(Task 8)
workflows/mable-job-hunter.json    匯出的 n8n workflow
docs/ops.md                        Mac mini 常駐設定
```

---

### Task 0: 在 Mac 上安裝並啟動 n8n(Docker)+ 連上 Google 帳號 + 建立 Telegram bot

**Files:**
- Create: `docker-compose.yml`(只有 `n8n` 一個 service)、`.env.example`、`.gitignore`(含 `.env`)、`docs/ops.md`(先寫「安裝與更新」一節)

**Interfaces:**
- Produces: n8n 在 `http://localhost:5678` 運行,資料在 named volume `n8n_data`,已建立可用的 `Gmail OAuth2`、`Google Calendar OAuth2` 與 `Telegram` credentials,並記下你的 Telegram chat ID。Task 8 會在同一個 compose 檔新增 `job-filter` service。

**為何選 Docker 而非 npm:** 之後要和評估服務放在同一個 compose 網路、升級與回滾簡單、不污染本機 Node。npm 安裝(`npm install -g n8n`,本機 Node 24 可用)是備案,但升級與常駐都要自己管。Docker Desktop 對初學者最直觀,因此選它;若日後想完全無頭運行再換 Colima。

此 task 是環境設定,沒有單元測試;每步以可檢查的輸出驗證。標示「使用者」的步驟需要你本人操作(密碼、瀏覽器登入)。

- [ ] **Step 1(使用者): 安裝 Docker Desktop**
  Run: `brew install --cask docker`,然後從「應用程式」開啟 Docker,依畫面同意條款並輸入 Mac 密碼授權。
  Run: `docker --version && docker compose version`
  Expected: 兩個指令都印出版本號(不再是 `command not found`)。

- [ ] **Step 2: 建立 `docker-compose.yml`(n8n 單一 service)與 `.env.example`**
  決定的值:image `n8nio/n8n`(釘到建立當天 stable 的具體版本號,不用 `latest`;文件顯示目前 stable 為 `2.41.7`,以建立時 docs 為準);`ports: ["127.0.0.1:5678:5678"]`;`environment` 含 `TZ=Australia/Brisbane`、`GENERIC_TIMEZONE=Australia/Brisbane`、`N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`;volume `n8n_data:/home/node/.n8n`;`restart: unless-stopped`。`.env.example` 先留空註解,Task 8 再填。

- [ ] **Step 3: 啟動並驗證**
  Run: `docker compose up -d && sleep 15 && curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5678/healthz`
  Expected: `200`。
  (使用者)用瀏覽器開 `http://localhost:5678`,建立 owner 帳號;**把 n8n 自動產生的加密金鑰備份**(存在 volume 內 `/home/node/.n8n/config`,建議另存密碼管理器,否則 volume 遺失後 credentials 無法還原)。

- [ ] **Step 4: 驗證資料可在重啟後保留**
  Run: `docker compose down && docker compose up -d`,重新登入 n8n。
  Expected: owner 帳號與剛建立的測試 workflow 仍在。

- [ ] **Step 5(使用者): 建立 Google OAuth 憑證**
  1. Google Cloud Console 建立專案,啟用 **Gmail API** 與 **Google Calendar API**。
  2. OAuth consent screen:User type 選 External;加入自己的 Gmail 為 test user;**發布狀態改為 In production**(個人使用、未驗證的警告頁按「進階 → 繼續」即可)。原因:External 且停在 *Testing* 狀態時,refresh token 約 7 天就會失效,workflow 會在一週後無聲失敗(請在此步驟用 Google 當下的文件再確認)。
  3. Credentials → OAuth client ID(Web application),Authorized redirect URI 填 `http://localhost:5678/rest/oauth2-credential/callback`。
  4. 在 n8n 新增 `Gmail OAuth2 API` 與 `Google Calendar OAuth2 API` credentials,貼入 Client ID / Secret,按 Connect 並授權。
  Expected: 兩個 credential 在 n8n 顯示 *Connected*;用一個臨時 workflow(Manual Trigger → Gmail *Get Many*,limit 1)能讀到一封信。

- [ ] **Step 6(使用者): 建立 Telegram bot**
  1. 在 Telegram 找 `@BotFather`,傳 `/newbot`,依提示取名,取得 **bot token**(不要貼給任何人或對話框,直接貼進 n8n)。
  2. 開啟你的新 bot 聊天並傳一句話(否則 bot 無法主動傳訊息給你)。
  3. 取得你的 chat ID:在 Telegram 找 `@userinfobot` 傳任意訊息,它會回傳 `Id`。
  4. 在 n8n 新增 `Telegram API` credential,貼入 bot token。
  Verify: 臨時 workflow 用 Manual Trigger → Telegram *Send Message*(Chat ID=你的 ID,文字 `[Job] test`)。
  Expected: 手機收到 bot 傳來的訊息並跳出通知。(只用「傳送」,不用 Telegram Trigger,所以不需要對外網的 HTTPS webhook。)

- [ ] **Step 7: Commit**
```bash
git init && git add docker-compose.yml .env.example .gitignore docs/ops.md && git commit -m "chore: run n8n locally via docker compose"
```

---

### Task 1: 專案骨架、共用型別、偏好設定載入 + 資料來源 Gate

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`, `.gitignore`, `src/types.ts`, `src/config.ts`, `config/preferences.json`
- Test: `tests/config.test.ts`

**Interfaces:**
- Produces(後續全部 task 依賴,名稱不得更動):
```ts
export type Weekday = 1 | 2 | 3 | 4 | 5 | 6 | 7;        // ISO,1=週一
export type Schedule =
  | { kind: 'fixed'; start: string; end: string }          // ISO 8601 含 offset
  | { kind: 'recurring'; weekday: Weekday; startTime: string; endTime: string } // "HH:mm";endTime<startTime 表示跨夜
  | { kind: 'flexible'; hoursPerVisit?: number };
export interface RawJob {
  id: string; url: string; title: string; description: string;
  categories: string[]; suburb: string; schedule: Schedule;
}
export interface Interval { start: string; end: string }   // ISO 8601
export interface CalendarEvent { summary?: string; start: string; end: string; allDay: boolean }
export type Outcome = 'pass' | 'fail' | 'unknown';
export interface RuleResult { outcome: Outcome; reason: string; flag?: string }
export type Verdict = 'match' | 'maybe' | 'reject';
export interface Decision { jobId: string; verdict: Verdict; results: Record<'category'|'duration'|'location'|'calendar', RuleResult> }
export interface Preferences {
  timezone: string; minDurationHours: number; horizonWeeks: number; travelBufferMinutes: number;
  categories: { allow: string[]; blockKeywords: string[] };
  suburbs: { anywhere: string[]; campusArea: string[] };
  campus: { days: Weekday[]; classEndsAt: string };
  allDayBlockKeywords: string[];
}
export function loadPreferences(path: string): Preferences;
```

- [ ] **Step 1: Gate — 確認資料來源(需使用者動手,先於所有程式碼)**
  請使用者:(a) 在 Mable 網站/App 的通知設定確認有「新工作提醒」郵件;(b) 把 3 封以上提醒郵件另存為 HTML 放到 `tests/fixtures/mable-alert-N.html`(刪除個人資料);(c) 用文字寫下每封的預期解析結果。
  通過條件:郵件內可看到標題、類型、郊區、時間(至少其中「類型 + 郊區」)。**若沒有提醒郵件或資訊不足,停止此 plan**,另開 plan 評估替代來源(例如手機通知轉發或使用者授權的瀏覽器擴充),不要硬爬。

- [ ] **Step 2: Write the failing test** `tests/config.test.ts`:
  - `loads a valid file`:載入 `config/preferences.json` 得到 `minDurationHours === 2`、`timezone === 'Australia/Brisbane'`、`campus.days` 等於 `[2, 4]`。
  - `normalizes suburb names`:檔內寫 `"  Surfers Paradise "` → `suburbs.anywhere` 含 `"surfers paradise"`。
  - `rejects bad values`:`minDurationHours: 0` 與 `classEndsAt: "25:99"` 各自 `toThrow()`。

- [ ] **Step 3: Run** `npx vitest run tests/config.test.ts` — Expected: FAIL(模組不存在)。

- [ ] **Step 4: Implement** 上述型別(`src/types.ts`)與 `loadPreferences`(`src/config.ts`,zod schema;郊區與類型字串 `trim().toLowerCase()`)。`config/preferences.json` 預設值以 Global Constraints 為準;`categories.allow` 先填 `["gardening","domestic assistance","cleaning"]`,`blockKeywords` 先填 `["sleepover","hoist"]`;`suburbs` 草稿由使用者逐一確認(G:link 沿線:Helensvale 至 Broadbeach South;火車線:Varsity Lakes 至 Helensvale;campusArea:Bilinga、Tugun、Coolangatta 一帶)。

- [ ] **Step 5: Run** `npx vitest run tests/config.test.ts` — Expected: PASS。

- [ ] **Step 6: Commit**
```bash
git add -A && git commit -m "chore: scaffold project, types and preferences loader"
```

---

### Task 2: Mable 提醒郵件解析器

**Files:**
- Create: `src/parse/mableEmail.ts`
- Test: `tests/parse/mableEmail.test.ts`, `tests/fixtures/mable-alert-*.html` + `*.expected.json`(Task 1 gate 取得)

**Interfaces:**
- Consumes: `RawJob`, `Schedule`(Task 1)
- Produces:
```ts
export class MableParseError extends Error {}
export function parseMableAlertEmail(html: string, timezone: string): RawJob[];
```
`id` 用工作連結中的 Mable job 識別碼(若郵件無此識別碼,則用連結的穩定雜湊);拋 `MableParseError` 條件:HTML 中找不到任何工作區塊。

- [ ] **Step 1: Write the failing tests**
  - `parses every fixture to its expected jobs`:對 `tests/fixtures` 下每個 `mable-alert-N.html`,`parseMableAlertEmail(html, 'Australia/Brisbane')` 深度等於 `mable-alert-N.expected.json`(golden file,內容來自使用者在 Task 1 寫的預期)。
  - `throws MableParseError when no jobs found`:輸入 `<html><body>hello</body></html>` → `toThrow(MableParseError)`。
  - `returns a flexible schedule when time text is vague`:fixture 中「時間彈性」的工作 `schedule.kind === 'flexible'`。

- [ ] **Step 2: Run** `npx vitest run tests/parse` — Expected: FAIL。

- [ ] **Step 3: Implement** `parseMableAlertEmail`(cheerio 選取工作區塊;時間文字轉成 `Schedule`:有具體日期 → `fixed`;「每週二 2–4pm」→ `recurring`;其他 → `flexible`)。選擇器與時間文字格式以 fixture 為準;若 fixture 顯示格式有多種,先支援 fixture 中出現的,未出現的一律落到 `flexible`,不要猜。

- [ ] **Step 4: Run** `npx vitest run tests/parse` — Expected: PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: parse Mable alert emails into RawJob"`

---

### Task 3: 類型與時長規則

**Files:**
- Create: `src/rules/category.ts`, `src/rules/duration.ts`
- Test: `tests/rules/category.test.ts`, `tests/rules/duration.test.ts`

**Interfaces:**
- Consumes: `RawJob`, `Preferences`, `RuleResult`
- Produces:
```ts
export function checkCategory(job: RawJob, prefs: Preferences): RuleResult;
export function checkDuration(job: RawJob, prefs: Preferences): RuleResult;
export function scheduleHours(schedule: Schedule): number | undefined; // flexible 且無 hoursPerVisit → undefined
```

- [ ] **Step 1: Write the failing tests**
  - category:`categories:['Gardening']` → `pass`;`categories:['Personal Care']`(不在 allow)→ `fail`;`categories:['Gardening']` 但描述含 `Sleepover` → `fail`(blockKeywords,不分大小寫)。
  - duration:`fixed 14:00–16:00` → `pass`(剛好 2h);`fixed 14:00–15:30` → `fail`;`recurring 22:00–06:00` → `scheduleHours === 8` 且 `pass`;`flexible` 無 `hoursPerVisit` → `unknown`;`flexible hoursPerVisit: 3` → `pass`。

- [ ] **Step 2: Run** `npx vitest run tests/rules/category.test.ts tests/rules/duration.test.ts` — Expected: FAIL。

- [ ] **Step 3: Implement** 以上三個函式(luxon 解析;`recurring` 的 `endTime <= startTime` 加 24 小時)。每個 `RuleResult.reason` 用中文一句話說明(例:`時長 1.5 小時 < 2 小時`)。

- [ ] **Step 4: Run** 同 Step 2 — Expected: PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: category and duration rules"`

---

### Task 4: 地點規則(含週二/週四校園區)

**Files:**
- Create: `src/rules/location.ts`
- Test: `tests/rules/location.test.ts`

**Interfaces:**
- Consumes: `RawJob`, `Preferences`, `RuleResult`
- Produces:
```ts
export function normalizeSuburb(raw: string): string;   // 小寫、去 "QLD"/郵遞區號/多餘空白
export function checkLocation(job: RawJob, prefs: Preferences): RuleResult;
```
規則:郊區 ∈ `suburbs.anywhere` → `pass`;∈ `suburbs.campusArea` → 排程為 `fixed`/`recurring`、Brisbane 星期 ∈ `campus.days`、且開始時間 ≥ `classEndsAt + travelBufferMinutes` 才 `pass`,否則 `fail`;排程為 `flexible` → `unknown`(`flag: 'campus-tue-thu-afternoon-only'`);都不在 → `fail`。

- [ ] **Step 1: Write the failing tests**
  - `normalizeSuburb('SURFERS PARADISE')`、`normalizeSuburb('Surfers  Paradise QLD 4217')` 都等於 `'surfers paradise'`。
  - 白名單郊區 → `pass`;名單外 → `fail`。
  - campusArea(Bilinga)+ `recurring 週二 14:30–17:00` → `pass`;`週二 13:30`(< 13:00+45min)→ `fail`;`週三 15:00` → `fail`;`flexible` → `unknown`。
  - 時區:campusArea 的 `fixed` 工作開始於 `2026-10-06T05:00:00Z`(= Brisbane 週二 15:00)→ `pass`;開始於 `2026-10-06T15:00:00Z`(UTC 仍是週二 15:00,但 = Brisbane 週三 01:00)→ `fail`。證明用的是 Brisbane 星期而非 UTC 星期。

- [ ] **Step 2: Run** `npx vitest run tests/rules/location.test.ts` — Expected: FAIL。

- [ ] **Step 3: Implement** 以上兩個函式。

- [ ] **Step 4: Run** 同 Step 2 — Expected: PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: location rule with Tue/Thu campus-area handling"`

---

### Task 5: 日曆規則

**Files:**
- Create: `src/rules/calendar.ts`
- Test: `tests/rules/calendar.test.ts`

**Interfaces:**
- Consumes: `RawJob`, `Preferences`, `CalendarEvent`, `Interval`, `RuleResult`
- Produces:
```ts
export function toBusyIntervals(events: CalendarEvent[], prefs: Preferences): Interval[];
export function occurrences(job: RawJob, now: string, prefs: Preferences): Interval[]; // flexible → []
export function checkCalendar(job: RawJob, busy: Interval[], now: string, prefs: Preferences): RuleResult;
```
規則:
- `toBusyIntervals`:非全天事件原樣轉換;全天事件只有 `summary` 含 `allDayBlockKeywords`(不分大小寫)才展開成該日 00:00–24:00(Brisbane),其餘忽略。
- `occurrences`:`fixed` → 自身(已過去則 `[]`);`recurring` → `now` 起 `horizonWeeks` 週內每週一次(Brisbane 星期,跨夜則 end 在次日)。
- `checkCalendar`:每個 occurrence 兩端各加 `travelBufferMinutes` 後與 busy 比對。`flexible` → `unknown`;`fixed` 已過去 → `fail`;全部衝突 → `fail`;部分衝突 → `unknown`(`flag` 列出衝突日期);沒有衝突 → `pass`。

- [ ] **Step 1: Write the failing tests**
  - `all-day 'Nursing Placement' blocks that day`、`all-day 'Mum birthday' is ignored`。
  - fixed 工作 15:00–17:00,日曆在 17:20 結束的課 → 因緩衝 45 分鐘而 `fail`;17:50 結束 → `pass`。
  - recurring 每週二 14:00–16:00,日曆只有一次週二衝突 → `unknown`;四週全衝突 → `fail`;無衝突 → `pass`。
  - `flexible` → `unknown`;`fixed` 在 `now` 之前 → `fail`。
  - 時區:`now = 2026-10-05T14:00:00Z`(Brisbane 週二 00:00 之後)時,`recurring` 週二的第一個 occurrence 落在 Brisbane 當天,不是下週。

- [ ] **Step 2: Run** `npx vitest run tests/rules/calendar.test.ts` — Expected: FAIL。

- [ ] **Step 3: Implement** 以上三個函式(luxon 於 `prefs.timezone` 內運算)。

- [ ] **Step 4: Run** 同 Step 2 — Expected: PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: calendar conflict rule with travel buffer"`

---

### Task 6: 評估引擎

**Files:**
- Create: `src/evaluate.ts`
- Test: `tests/evaluate.test.ts`

**Interfaces:**
- Consumes: Task 3–5 的四個 `check*` 函式
- Produces:
```ts
export interface EvalContext { prefs: Preferences; busy: Interval[]; now: string }
export function evaluateJob(job: RawJob, ctx: EvalContext): Decision;
```
四條規則一律執行(不短路);任一 `fail` → `reject`;否則任一 `unknown` → `maybe`;否則 `match`。

- [ ] **Step 1: Write the failing tests**
  - 四條全 pass → `match`。
  - 類型 fail + 其他 pass → `reject`,且 `results.category.outcome === 'fail'`、其餘三項結果仍存在。
  - `flexible` 工作其他皆 pass → `maybe`。
  - 同時有 `fail` 與 `unknown` → `reject`。

- [ ] **Step 2: Run** `npx vitest run tests/evaluate.test.ts` — Expected: FAIL。

- [ ] **Step 3: Implement** `evaluateJob`。

- [ ] **Step 4: Run** `npx vitest run` — Expected: 全部 PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: combine rules into evaluateJob"`

---

### Task 7: 去重儲存 + HTTP 服務

**Files:**
- Create: `src/store.ts`, `src/server.ts`, `src/main.ts`
- Test: `tests/store.test.ts`, `tests/server.test.ts`

**Interfaces:**
- Consumes: `parseMableAlertEmail`, `MableParseError`, `evaluateJob`, `toBusyIntervals`, `loadPreferences`
- Produces:
```ts
export interface Store {
  isSeen(jobId: string): boolean;
  markSeen(entries: { jobId: string; verdict: Verdict }[], at: string): void;
  stats(since: string): { total: number; match: number; maybe: number; reject: number };
  close(): void;
}
export function openStore(path: string): Store;          // ':memory:' 可用於測試
export function buildServer(deps: { prefs: Preferences; store: Store; now: () => string }): FastifyInstance;
```
HTTP 契約:
- `GET /health` → `200 {ok:true}`
- `POST /parse {html}` → `200 {jobs: RawJob[]}`;`MableParseError` → `422 {error:'parse_failed'}`
- `POST /process {jobs: RawJob[], events: CalendarEvent[]}` → `200 {decisions: (Decision & {job: RawJob})[]}`,**只含未讀工作**,且此呼叫不寫入已讀
- `POST /seen {entries: {jobId, verdict}[]}` → `200 {ok:true}`
- `GET /stats?since=ISO` → `200 {total, match, maybe, reject}`

- [ ] **Step 1: Write the failing tests**
  - store:`markSeen` 後 `isSeen` 為 true;`stats` 依 verdict 計數且只計 `since` 之後。
  - server(`app.inject`):`/health` 200;`/parse` 對 fixture 回 jobs,對 `<html></html>` 回 422;`/process` 對同一 job 在 `/seen` 之前呼叫兩次都回該 job,`/seen` 之後回空陣列;同一批次內重複 `id` 只出現一次(對應「同一工作出現在多封郵件」)。

- [ ] **Step 2: Run** `npx vitest run tests/store.test.ts tests/server.test.ts` — Expected: FAIL。

- [ ] **Step 3: Implement** `openStore`(better-sqlite3,表 `seen_jobs(job_id PRIMARY KEY, verdict, seen_at)`)、`buildServer`、`main.ts`(讀 `PREFS_PATH`、`DB_PATH`、`PORT` 環境變數,預設 `3000`)。

- [ ] **Step 4: Run** `npx vitest run` — Expected: 全部 PASS。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: dedup store and HTTP service"`

---

### Task 8: job-filter 容器 + n8n workflow + Telegram 通知

**Files:**
- Create: `Dockerfile`, `docker-compose.yml`, `.env.example`, `workflows/mable-job-hunter.json`, `docs/ops.md`

**Interfaces:**
- Consumes: Task 7 的 HTTP 契約。服務在 compose 網路內位址 `http://job-filter:3000`。

此 task 是設定與接線,沒有單元測試;以下以 dry-run 驗證。

- [ ] **Step 1: 建立 `Dockerfile`(評估服務,base image 的 Node major 與本機一致)並在 Task 0 的 `docker-compose.yml` 新增 `job-filter` service**:`TZ=Australia/Brisbane`、`restart: unless-stopped`;SQLite 用獨立 named volume;不對主機開放埠(只在 compose 網路內讓 n8n 以 `http://job-filter:3000` 存取,除錯時才臨時映射)。

- [ ] **Step 2: Run** `docker compose up -d --build && docker compose exec n8n wget -qO- http://job-filter:3000/health`
  Expected: `{"ok":true}`(同時證明 n8n 容器能連到 job-filter)。

- [ ] **Step 3: 在 n8n UI 建立 workflow 並匯出到 `workflows/mable-job-hunter.json`**,節點順序:
  1. Gmail Trigger(輪詢每 5 分鐘;篩選 `from:mable.com.au`)
  2. HTTP Request `POST /parse`(body `{html}`;設定「錯誤時繼續」)
  3. IF `/parse` 回 422 → Telegram「⚠️ Mable 郵件解析失敗」附原郵件主旨(Review Focus #1)
  4. Google Calendar:Get Many events,未來 4 週,所有相關日曆
  5. Code node:把事件整理成 `CalendarEvent[]`(`allDay = 無 dateTime`)
  6. HTTP Request `POST /process`
  7. Split Out → IF `verdict !== 'reject'` → Telegram *Send Message*:**每個工作一則訊息**(點連結直接開 Mable),內容為 `✅ match` 或 `🤔 maybe` + 標題、郊區、時間、連結,`maybe` 再附「哪條規則是 unknown」的原因。Parse Mode 不設(純文字),避免標題含 `_`、`*` 導致 Telegram 拒收;不要把工作描述全文放進訊息(Telegram 單則上限 4096 字)。
  8. HTTP Request `POST /seen`(**只接在 Telegram 傳送成功之後**;reject 的工作也要標記)

- [ ] **Step 4: Dry-run** 把一封 fixture 郵件貼進手動觸發的版本執行。
  Expected: 符合條件的工作各自收到一則 Telegram 訊息;再執行一次,**不再重複推送**;故意傳入空 HTML,收到「解析失敗」訊息;把 Telegram 節點的 chat ID 故意改錯再執行,`/seen` **不會**被呼叫(驗證「傳送失敗不標記已讀」,下次才不會漏掉工作)。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: docker compose and n8n workflow"`

---

### Task 9: 常駐運行與靜默失效警報

**Files:**
- Modify: `workflows/mable-job-hunter.json`(新增兩個流程)
- Modify: `docs/ops.md`

- [ ] **Step 1: 每日摘要 workflow**:Schedule Trigger 每天 20:00 → `GET /stats?since=<24 小時前>` → Telegram「📊 每日摘要:今天處理 N 個、match M、maybe K」。若連續 3 天 `total === 0` 則加註「⚠️ 可能收不到 Mable 郵件了」。

- [ ] **Step 2: 錯誤 workflow**:n8n「Error Trigger」→ Telegram「⚠️ workflow 失敗:<節點名稱>」;在主 workflow 設定使用它。Telegram 與 Google 憑證互相獨立,Google 授權過期也能收到警報;但若「每日摘要沒收到」,代表 n8n 或 Mac mini 本身停了,`docs/ops.md` 要寫明這個判斷方式。

- [ ] **Step 3: 寫 `docs/ops.md` 並在 Mac mini 實際設定**:
  - `sudo pmset -a sleep 0 autorestart 1`(不睡眠、停電後自動開機);
  - Docker Desktop 設為登入時啟動,Mac 設為自動登入(或改用 Colima + launchd);
  - 備份:每週複製 n8n volume 與 SQLite 檔。

- [ ] **Step 4: 驗證** 重新開機 Mac mini,不登入操作。
  Expected: `docker compose ps` 兩個服務皆 `running`;隔天 20:00 收到每日摘要。

- [ ] **Step 5: Commit** `git add -A && git commit -m "feat: daily digest, error alerts, ops notes"`

---

## Self-Review

- **Spec 涵蓋**:n8n 本機安裝、Google 授權與 Telegram bot(T0)、類型(T3)、時長(T3)、地點含週二四校園區(T4)、日曆(T5)、去重(T7)、Telegram 推送(T8)、24/7 與靜默失效(T8–9)。四條規則的判定組合在 T6。
- **步驟掃描**:程式碼步驟皆給出簽名與檔案,沒有寫出函式本體。T2 的選擇器與時間文字格式取決於使用者提供的 fixture,已在 Task 1 設為 gate。
- **型別一致**:`RawJob`、`Schedule`、`Interval`、`CalendarEvent`、`RuleResult`、`Decision`、`Preferences` 於 T1 定義,T2–T7 引用的名稱一致;`checkCalendar(job, busy, now, prefs)` 參數順序在 T5 與 T6 一致。
- **Review Focus**:五項各有對應測試(#1: T2/T7/T8、#2: T5、#3: T3、#4: T4、#5: T4/T5)。
- **比例**:plan 約為 spec 的 4–5 倍,主要是因為測試案例與 n8n 節點清單,內容皆為 spec 未決定的細節。
