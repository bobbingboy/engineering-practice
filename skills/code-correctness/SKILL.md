---
name: code-correctness
description: 功能完成後的 correctness 與 reuse 審查——找會造成錯誤行為的 bug，以及「共用層已經有了卻又寫一份」的重造，由獨立 sub-agent 執行以避免「自己 review 自己」的盲點。語言中性，涵蓋前端 JS 與後端 Java。Use when 功能完成後要找 bug、改動觸及共用層或既有機制、或被 /finalize 串接呼叫。
argument-hint: "[檔案路徑、目錄或 commit hash，省略則取 git diff] [focus=contract|reachability|boundary|all]"
---

# code-correctness

對 **$ARGUMENTS** 進行 correctness 審查。

**核心策略**：由獨立 sub-agent 在乾淨 context 中執行 review，主 session 只負責蒐集脈絡、過濾報告、套用修復。這避免「寫 code 的人 = review 的人」造成的合理化偏誤。

## 與姊妹 skill 的分工

| 檢查項目 | code-quality | react-perf-check | **code-correctness** |
|---|---|---|---|
| 命名 / 清理 / 結構硬指標 | ✅ | ❌ | ❌ |
| useEffect 依賴 / memo / Context 穩定性 | ❌ | ✅ | ❌ |
| 會造成錯誤行為的 bug | ❌ | ❌ | ✅ |
| 與既有機制的互動 | ❌ | ❌ | ✅ |
| **已有實作被重造（reuse）** | ❌ | ❌ | ✅ |
| 後端 Java | ❌ | ❌ | ✅ |

三者互補非重疊。correctness 是三者中唯一需要**讀 diff 以外的程式碼**才能做的，reuse 因此掛在這裡——判斷「這個 codebase 已經有了」需要的正是同一批閱讀。`code-quality` 的 rubric 明文不管跨檔案的重複，所以這裡不報就沒有人報了。

## 適用場景

- 功能開發完成後找 bug
- 改動觸及共用 lib（`src/lib/`）、核心模組或任何被多處呼叫的共用層
- 修正了某個錯誤，想知道它讓什麼變成可達
- 被 `/finalize` skill 串接呼叫（階段 2b）

## focus 參數

預設 `all`：`contract`（契約破壞）/ `reachability`（新增可達性）/ `boundary`（邊界與空值）/ `all`。

## 執行流程

### Step 1：計算審查範圍

- **指定 commit hash** → `git show <hash> --stat` 取檔案清單
- **指定路徑** → 該路徑下的原始碼檔（`.js/.jsx/.ts/.tsx/.java`）
- **未指定** → 取 git 變更：

  ```bash
  git diff --name-only
  git diff --cached --name-only
  git diff -U0                       # 行號 hunks
  ```

**不要**過濾掉 `.java`——這是本 skill 與 `code-quality` 的關鍵差別。也不要過濾掉 `*.test.js`，測試本身可能是錯的，或漏掉了改動的關鍵路徑。

### Step 2：蒐集「不要質疑」脈絡

這一步是降低 false positive 的關鍵，也是外部 review 工具做不到的部分。組裝：

1. **OpenSpec 設計脈絡**（若有）：
   - 偵測 `openspec/changes/<name>/` 是否有對應本次變更的活躍 change
   - 讀 `proposal.md` 與 `design.md`，萃取 1–3 句「設計決策摘要」
2. **專案規範摘要**：
   - `modules/CLAUDE.md` 與相關子模組 CLAUDE.md 中，本次 diff 觸及目錄的 sync 規則、「不同步」清單
   - **共用 lib 修改門檻**：若 diff 迴避了修改共用 lib 而採用擴充點（interceptor、包一層），這是刻意的，不要建議搬回 lib
3. **本次對話的決策**：主 session 已明確討論並決定的事項

若全無內容，傳「無特殊脈絡」，**不要**捏造。

### Step 3：讀取 rubric

讀同目錄的 `CORRECTNESS_RUBRIC.md` 全文，準備併入 sub-agent prompt。

### Step 4：Spawn 審查 sub-agent

呼叫 Agent，參數：

- `subagent_type: "claude"`（**不是** Explore——correctness 需要讀大量 diff 以外的實作，且可能要跑測試驗證推論）
- `description: "Correctness review {功能名稱}"`
- `prompt:` 依下方範本

**Prompt 範本**：

```
你是獨立的 code reviewer，做 correctness 導向的審查。工作目錄 {cwd}。

## 操作提示
**不要**一次執行 `git show <hash>` 或 `git diff` 取全部內容——大改動的單次輸出會讓 agent 卡死（實測 1686 行插入的 commit 直接 stall 掉整個 review）。逐檔取：

```
git show <hash> --stat --format=''       # 先看檔案清單
git show <hash> -- <單一路徑>             # 一次一個檔案
git show <hash>:<路徑> | sed -n '起,迄p'  # 需要全文脈絡時分段讀
```

## 審查範圍
focus: {focus 值}
{檔案與行號 hunks，或 commit hash}

## 功能脈絡
{一段話說明這次改動要解決什麼問題，讓 reviewer 判斷得出「有沒有解到」}

## 設計脈絡（已決策事項，不要質疑）
{Step 2 蒐集的內容；若無，寫「無特殊脈絡」}

## Rubric
{CORRECTNESS_RUBRIC.md 全文}

## 你必須實際去讀的東西
不要只讀 diff。至少去讀：
- diff 呼叫到的共用實作（共用 lib、核心模組的 service/repository）
- 新丟出的 error／新回傳值的呼叫端
- 同一份檔案在其他模組的副本（office / portal / expo）

讀這些的時候一併回答：**diff 新寫的東西，這些既有實作是不是已經在做了？**（rubric 1.2）

{若有可跑的測試，補上指令讓 reviewer 自行驗證推論}
```

### Step 5：主 session 過濾報告

| 過濾規則 | 動作 |
| --- | --- |
| 無「觸發條件 → 後果」，只有「可能會…」 | 丟 |
| 無佐證（未讀過就斷言共用函式行為） | 丟 |
| 無明確「檔案:行號」 | 丟 |
| 與「設計脈絡」明確衝突（質疑已決策事項） | 丟 |
| 指責下游客製專案的專屬目錄、共用 lib 的同步邏輯 | 丟 |
| 落在 diff 行號範圍外，且未說明與 diff 的關聯 | 丟 |
| 屬 `code-quality` 範圍（命名、未使用 import、風格） | 轉交，不在此處呈現 |
| 其餘 | 保留 |

**嚴重度自行複核**，不要照抄 sub-agent 的標籤——實測會有把「使用者永久失去重新登入路徑」評成 medium 的情況。判準見 rubric 第 4 段。

### Step 6：呈現裁決

| 級別 | 預設處理 |
| --- | --- |
| 🔴 critical / high | 預設採納，除非使用者反對 |
| 🟡 medium | 列出來問使用者要不要修 |
| 🟢 low | 僅告知 |

**同根的 finding 合併成一筆**，標明它有幾個後果。實測同一個根因會被拆成三筆分別呈現，讓人誤以為是三個獨立問題。

### Step 7：套用修復

對使用者確認的項目，**主 session 直接用 Edit 修改**。不再呼叫 sub-agent——修復必須在主 context 內，才能配合後續 commit / `/finalize` 流程。

每個修改都讀檔再改，不要從記憶套。改完跑測試；若改動的是共用層或有多份副本，確認副本一致。

## 注意事項

- **sub-agent 用 `claude` 不用 `Explore`**：correctness 需要讀 diff 以外的大量實作，read-only 的搜尋型 agent 會停在表面
- **不要在 prompt 裡暗示答案**。若主 session 已經知道某個 bug，不要寫進 prompt——那會讓你分不清 reviewer 是自己找到的還是被引導的
- 一次審查範圍不宜過大；超過 ~15 個檔案時依功能切分多次呼叫
- 若改動含資料庫 schema，另外確認有無對應 migration（見 CLAUDE.md 的 Migration 規範）
