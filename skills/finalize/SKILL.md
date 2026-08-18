---
name: finalize
description: 功能實作完成後的收尾流程：依序歸檔 OpenSpec change（若適用）、執行 code-quality 與 code-correctness、執行 react-perf-check（僅當改動含 React 前端程式碼）、套用修復、commit（嚴格的檔案範圍控制 + 留意檔案警告）、最後在明確授權下 push。當使用者表達「收尾」、「finalize」、「功能完成了」、「可以送出」、「準備 commit」、「ready to ship」、「一起處理完」、「把這個功能結掉」、或任何表示「單一功能已實作完畢、要走完最後幾步」的語句時觸發。優先使用本 skill，而非分別手動呼叫 /opsx:archive、/code-quality、/react-perf-check、/commit。
argument-hint: "[可選：OpenSpec change 名稱] [--skip-e2e|--skip-archive|--skip-review|--skip-perf|--no-push|--second-opinion]"
---

# finalize

功能實作完成後的收尾流程工具。**不是每次 commit 都跑**——這是「一個功能走到終點時」才用。

## 為什麼要一個 skill

收尾工作有多個步驟，**順序有講究**（review 必須在 commit 前才能把修復一起帶入 commit；歸檔要在 commit 前才能讓產生的 OpenSpec 檔案一併提交），而且有**若干高風險點**（push 是共享狀態、留意檔案容易誤 commit、review findings 不能盲目全套用）。一個人工每次從頭跑一遍容易漏步驟或誤操作，所以把流程和護欄一起 encode 進 skill。

## 階段總覽

```
0. 偵測       → 列狀態、決定要跑哪些階段
0.5 端到端驗證 → 改動落入單元測試盲區時，起環境實測（內建 `run` skill，或專案自有的環境 skill）
1. 歸檔       → opsx:archive（若有活躍 change）
2a. code-quality → 機械式品質（命名/清理/結構硬指標，前端 JS）→ 裁決 → 套用
2b. code-correctness → correctness bug + reuse + 後端 Java 覆蓋（獨立 sub-agent + 專案脈絡）→ 裁決 → 套用
2c. 第二意見    → 僅 --second-opinion：獨立 reviewer 換一套規則集重審 → 裁決 → 套用
2.5 i18n 檢查 → 改動含翻譯檔或新 i18n 呼叫時，檢查 key 撞名與多語系同步
3. perf-check → 僅前端改動時跑 + 裁決 + 修復
4. commit 範圍確認 → 列擬 commit / 擬排除清單，等授權
5. commit    → git add 特定檔案 + git commit
6. push      → 明確授權後才執行
```

每個階段都有**可能卡住的點**，卡住就停下來讓使用者決定，不要猜。

## 階段 0：偵測

先做成本極低的偵測，讓使用者看到完整 picture 再決定要不要繼續。

```bash
openspec list --json 2>/dev/null   # 有哪些活躍 change
git status                          # 工作區狀態
git diff --stat                     # 改動哪些檔案
git diff --name-only                # 精確路徑（判斷前端/後端）
```

組合結果，列給使用者（範例）：

```
偵測結果：
- OpenSpec change:  add-collapsible-pipeline-header（artifacts 完整）
- 改動檔案:         4 個前端、0 個後端、2 個 i18n
- 留意檔案警告:     web/src/conf/config.js 有改動（請單獨處理）
- 無關變動:         portal/package.json 等 9 個（會預設排除）
- 測試產物:         collapsed-state.png 等 3 個（會預設排除）
- 端到端驗證:       需要（動到 repository 查詢條件、新增元件）／不需要（純重構）
- 建議流程:         端到端驗證 → archive → code-quality → react-perf-check → commit → (push 待授權)

要繼續嗎？若要調整（跳過某階段、改變範圍）請告訴我。
```

**等使用者點頭，不要自己跳到下一步。**

偵測階段就決定是否跳過以下階段：

| 條件 | 跳過階段 |
|------|----------|
| diff 不落入下方五類盲區任一類 | 階段 0.5 |
| 帶 `--skip-e2e` | 階段 0.5 |
| 無活躍 OpenSpec change 且未帶 change 參數 | 階段 1 |
| 帶 `--skip-archive` | 階段 1 |
| diff 全無 `.jsx/.tsx`，且 `.js` 皆不含 `use[A-Z]` hook | 階段 3 |
| diff 無 `src/nls/` 變更，且無新增 `i18n.translate(` / `t(` 呼叫 | 階段 2.5 |
| 帶 `--skip-review` | 階段 2a + 2b + 2c（所有審查都跳過） |
| **未**帶 `--second-opinion` | 階段 2c（預設不跑） |
| 帶 `--skip-perf` / `--skip-i18n` | 對應階段 |
| 帶 `--no-push` | 階段 6 僅告知不執行 |

## 階段 0.5：端到端驗證（條件）

單元測試全綠不代表功能正確。**改動落入以下任一類，就起環境實測**——這五類是單元測試的結構性盲區，不是「保險起見多跑一次」：

| 盲區 | 從 diff 判斷 | 單元測試為何看不到 |
|---|---|---|
| 查詢條件轉 SQL | repository、`Predicate`、JPQL／Criteria | mock 只驗得出 predicate 提到哪些欄位，驗不到資料庫實際回什麼 |
| 交易邊界 | `@Transactional`、initializer、跨 service 寫入 | 測試裡交易被獨立管理，跨方法的 rollback-only 傳播看不見 |
| 視覺與佈局 | 新元件、樣式或版面改動 | 元件邏輯測得到，CSS 在真實容器裡的行為測不到 |
| 初始化與資料狀態相依 | initializer、entity、migration、預設值 | 既有環境的 `count == 0` 分支根本不會進入 |
| 跨層串接 | controller 與前端 service 同時改、新增 API 欄位 | 兩端各自的測試都綠，中間的序列化行為沒人驗 |

跑法見內建 `run` skill，或專案自有的環境 skill。要點：選對環境（真實資料 vs 乾淨環境）、以功能實際使用者的角色身分操作、需要時直接查資料庫驗證、收尾還原環境。

**驗完要留下可查證的記錄**——寫進 change 的 tasks 或 commit message，內容是**實際看到的數字與帳號**，不是「已驗證」三個字。空的勾選沒有價值，日後也無從判斷當初驗了什麼。

**卡住點：**

| 情況 | 處置 |
|---|---|
| 落入盲區但使用者想跳過 | 問一句理由並記下，不要默默跳過 |
| 環境起不來 | 停下來報告卡在哪，不要為了繼續流程而略過驗證 |
| 需要寫入共用環境（SIT） | 先量原始狀態，驗完刪除並比對，確認回到原狀 |

> ⚠️ 若後續階段 2 的 review 修復**觸及行為**（而非命名／清理等品質性改動），回到本階段重驗。

## 階段 1：歸檔 OpenSpec change

透過 Skill 工具呼叫 `opsx:archive`，傳入 change 名稱。歸檔會：
- 同步 delta spec 至 `openspec/specs/<capability>/spec.md`
- 移動 change 資料夾至 `openspec/changes/archive/YYYY-MM-DD-<name>/`

**若 archive 失敗（target 目錄已存在、spec 衝突等）→ 停住**，不要繼續後面步驟。讓使用者處理檔案衝突再重跑。

歸檔產生的新檔案會一併進入稍後的 commit。

## 階段 2：程式碼審查（2a + 2b）

兩種審查抓的是**不同缺陷類別**，互補非重疊：

| 子階段 | 工具 | 抓什麼 | 範圍 |
|--------|------|--------|------|
| 2a | `code-quality` | 機械式品質：命名慣例、magic number、未用 import、console 殘留、函數過長、dead code | 前端 `.js/.jsx/.ts/.tsx` |
| 2b | `code-correctness` | 會造成錯誤行為的 bug、與既有機制的互動、**已有實作被重造（reuse）** | 語言中性，**含後端 Java** |

兩階段的 findings 都統一進下方三級裁決表，由**主 session** 套用修復：

| 級別 | 預設處理 |
|------|----------|
| 🔴 建議修復 / 確認 bug | 預設採納，除非使用者反對 |
| 🟡 建議改善 / 待確認 | 列出來問使用者要不要修 |
| 🟢 提醒 | 僅告知，不修改 |

### 2a：code-quality

透過 Skill 工具呼叫 `code-quality`。回報 findings 後依上表裁決。

### 2b：code-correctness

透過 Skill 工具呼叫 `code-correctness`，傳入本次改動範圍。它會蒐集 OpenSpec 決策與 CLAUDE.md 規則當作「不要質疑」脈絡、派獨立 sub-agent、依 `CORRECTNESS_RUBRIC.md` 審查，並自行做第一層過濾。

**不要改用內建的 `/code-review`。** 它標記為 `disable-model-invocation`，Skill 工具叫不動（會直接報錯），只能由使用者自己輸入 `/code-review` 觸發。若使用者想用它，請他自己下指令，不要在流程中等它。

`code-correctness` 回報後仍由主 session 做最終裁決，特別注意兩點（該 skill 的 Step 5/6 已載明，這裡重申因為容易漏）：
- **嚴重度自行複核**，不要照抄標籤
- **同根的 finding 合併成一筆**，標明它有幾個後果

### 2c：第二意見（僅 `--second-opinion`）

**存在理由**：2026-07-31 用歷史 bugfix 做過盲測回測——同一個 model、同樣的乾淨 context，換一套 checklist 得到的是「換一批 finding」而非「更多 finding」。三題中 2a+2b 漏掉一題（`b2d3666c3` 的 `Promise.all` 無錯誤隔離），換規則集那組抓到了；反之亦然。**增益來自第二個獨立 reviewer，不是來自哪套規則比較好。**

派**一個獨立 sub-agent**（不可在主 session 跑，否則失去獨立性），要它嚴格照 `~/.agents/skills/open-code-review-delegate/SKILL.md` 走：

```
ocr delegate preview            # 或 -c <hash> / --from --to
ocr delegate rule <paths...>
```

並明確指示它**只依 OCR 給的 rule 審查**，不要讀本專案的其他 skill 文件——重點是換一套規則集，混用就退化成重跑一次 2a。

裁決時對這組 findings 額外做兩件事：

- **嚴重度自行重評，不採信它的標籤**。回測中它兩次命中正解都只評 medium/low，照抄會在三級裁決表被當雜訊丟掉。
- **它看不到的檔案要自己補**：OCR 排除 `.md`（OpenSpec artifacts 整批不審）與 `*.test.js`。排除 deleted file 是正確行為，不用補。

與 2a/2b 重疊的 finding 合併成一筆，不要重複呈現給使用者。

**共同約束**：三個階段都**不要全部自動套用**。彙整裁決後逐項 Edit，每個修改都讀檔再改，不要從記憶套。若 2a 與 2b 指向同一行，合併成一筆處理。

## 階段 2.5：i18n 翻譯檢查（條件）

若階段 0 判定要跑（diff 含翻譯檔或程式碼新增 `i18n.translate(` / `t(` 呼叫），檢查：
- 是否有**既有 key 的 value 被修改**（高風險，污染全站）
- 是否引入了通用單字 key（如 `name`、`status`），應改用功能語境化 key
- zh / en 等語系是否同步
- 程式碼用到的 key 與翻譯檔是否一一對應

裁決邏輯同階段 2：findings 分級後逐項套用，不全自動。

若本階段未觸發，明確告知「因無 nls/ 改動，跳過 i18n 檢查」。

## 階段 3：react-perf-check（條件）

若階段 0 判定要跑，透過 Skill 工具呼叫 `react-perf-check`。裁決與套用邏輯同階段 2。

若本階段未觸發，明確告知「因無 React hook 改動，跳過 perf-check」。

## 階段 4：commit 範圍確認

這階段是整個流程最容易出錯的地方——**任何「一次 add 全部」的捷徑都是地雷**。

### 4.1 讀取留意檔案清單

讀取**當前專案**的 `CLAUDE.md`（或往上找最近的一個），從中找「Commit 留意檔案清單」或「留意檔案」章節，解析表格中列出的檔案路徑 / glob。

若找不到，使用預設：
- `src/conf/config.js`（任何模組底下）
- `*.properties`
- `.env`、`.env.*`
- `*secret*`、`*credential*`

### 4.2 分類

把 `git status` 列出的改動分三類：

**A. 擬 commit**：
- 本功能直接相關的程式碼檔案（從 `git diff` + 使用者對談脈絡推斷）
- OpenSpec 歸檔產生的新檔案（`openspec/changes/archive/<today>-*/` 與 `openspec/specs/<new-capability>/`）
- i18n 翻譯檔

**B. 留意 / 排除**：
- 任何命中 4.1 清單的檔案
- 與本功能明顯無關的 in-flight 變更（如 package.json 升版、不相關模組的改動）
- 測試產物：`.playwright-mcp/`、`*.png`（非 asset）、`*.log`

**C. 無法判斷**：
- 不確定相關性的檔案 → 直接問使用者

### 4.3 呈現 + 確認

以清單形式列出：

```
將 commit 的檔案：
  M  modules/web/src/.../OrderListHeader.js
  M  modules/web/src/.../OrderListContent.js
  M  modules/web/src/nls/message/zh.js
  M  modules/web/src/nls/message/en.js
  A  modules/openspec/changes/archive/2026-04-17-.../
  A  modules/openspec/specs/requisition-pipeline-header/

排除的檔案（留意檔案）：
  M  modules/web/src/conf/config.js  [CLAUDE.md 標記為留意]

排除的檔案（無關變動）：
  M  modules/portal/package.json
  M  modules/server/.../application.properties

排除的檔案（測試產物）：
  ?? .playwright-mcp/
  ?? collapsed-state.png
  ?? expanded-state.png

要繼續 commit 嗎？要調整範圍請告訴我。
```

**等使用者確認**。若使用者回「調整」，提供新的範圍；若回「好」，進階段 5。

## 階段 5：commit

### 5.1 stage

**只**用列出的明確路徑 `git add`：

```bash
git add <path1> <path2> <path3> ...
```

**禁止**：
- `git add -A`
- `git add .`
- `git add *`

這些會把未預期的檔案帶進來，違反階段 4 的 scope 控制。

### 5.2 message

若使用者有單獨的 `commit` skill，透過 Skill 工具呼叫它（它了解團隊規範）。否則依團隊慣例：

- 格式：`類型: 摘要`，類型常見為 `Feature` / `Bugfix` / `Refactoring` / `Docs`
- 繁體中文摘要
- 以 `Co-Authored-By: Claude ...` 結尾

用 HEREDOC 傳遞 message 以保留換行格式。

### 5.3 pre-commit hook 失敗處理

若 hook 阻擋，**不要 `--amend`**。commit 失敗代表 commit 尚未建立，`--amend` 會動到上一個 commit（錯誤且危險）。

正確流程：
1. 讀 hook 輸出，定位根因
2. 修 bug，再次 `git add` 相同檔案（如有新修改）
3. 重建新的 commit

### 5.4 回報

commit 成功後列出 hash + 摘要：

```
Commit fc4d1d5f3: Feature: ...
10 files changed, 433 insertions(+), 11 deletions(-)
```

## 階段 6：push

**這一步絕不能自動執行**。

commit 完成後告訴使用者：

```
待你確認後 `git push`（或直接回「push」我幫你跑）。
```

使用者明確說 `push` / `推` / `送上去` 才執行：

```bash
git push origin <current-branch>
```

### 特殊情境

| 情境 | 處理 |
|------|------|
| 分支為 `main` / `master` 且遠端拒絕 non-fast-forward | 停，告知使用者，建議 `git pull --rebase` |
| 需要 `--force` | 絕不自動加，必須使用者明確說「強推」 |
| 沒有 upstream | 執行 `git push -u origin <branch>`（這是安全的初次 push） |

## 禁忌清單

1. **不要在無授權下 `git push`**
2. **不要 `git commit --amend`** 除非使用者明確要求
3. **不要 `git add -A` / `git add .`**
4. **不要 `git reset --hard` / `git checkout --` / `git clean -f`**
5. **不要 `--no-verify` 跳過 hook**
6. **不要把 `src/conf/config.js` 類的留意檔案默默加進 commit**
7. **不要因為流程追求「一次走完」就跳過使用者確認點**——confirm 很便宜，誤操作很貴

## 失敗情境總表

| 階段 | 可能失敗 | 處理 |
|------|----------|------|
| 0 | 無 git repo | 告知並結束 |
| 1 | archive 目錄已存在 | 停，讓使用者手動處理 |
| 1 | spec 衝突 | 停，顯示差異讓使用者決定 |
| 2a/2b/2c | review 指出無法自動修的設計問題 | 停，詢問使用者怎麼判斷 |
| 2b | findings 誤指已決策事項／客製目錄 | 依脈絡過濾掉，不呈現給使用者 |
| 5 | pre-commit hook 失敗 | 修根因，建**新** commit |
| 5 | no changes to commit | 告知使用者可能 scope 過於嚴格，詢問 |
| 6 | remote rejected | 停，告知使用者需 pull/rebase |

## 範例對話

**User**: /finalize add-collapsible-pipeline-header

**Skill**: 執行階段 0，列出偵測結果，等待確認。

**User**: 好

**Skill**: 依序執行 1 → 2 → 3 → 4，在每個有 findings / 範圍確認點停下。

**User**: (依序回應：「採納」、「只修第 1 個」、「好」)

**Skill**: 執行階段 5，列出 commit hash，然後停在階段 6 前等待 push 授權。

**User**: push

**Skill**: 執行 `git push`，回報結果，結束。

## 不要做的事

- **不要展示工作細節給使用者**——中間的 Edit / Bash / Read 呼叫要照常跑，但進度更新要精簡。只在階段切換或需要裁決時與使用者互動。
- **不要「順便」修改與本功能無關的檔案**，即使看到明顯的 lint warning 或 typo。那些應該是另一個 commit。
- **不要假設使用者要 push**——就算已經 commit 過 100 次，每次還是要問。
