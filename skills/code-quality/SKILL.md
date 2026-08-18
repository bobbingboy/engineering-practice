---
name: code-quality
description: 功能完成後的程式碼審查，由獨立 sub-agent 執行以避免「自己 review 自己」的盲點。檢查簡化冗餘、命名規範、結構優化、清理。Use when 功能完成後的 code review、PR 前自審、或被 /finalize 串接呼叫。
argument-hint: "[檔案路徑或目錄，省略則取 git diff] [focus=simplify|naming|structure|cleanup|all]"
---

# code-quality

對 **$ARGUMENTS** 進行程式碼審查。

**核心策略**：由獨立 sub-agent 在乾淨 context 中執行 review，主 session 只負責蒐集脈絡、過濾報告、套用修復。這避免「寫 code 的人 = review 的人」造成的合理化偏誤。

## 適用場景

- 功能開發完成後的 Code Review
- PR 提交前的自我檢查
- 被 `/finalize` skill 串接呼叫

## focus 參數

預設 `all`：`simplify` / `naming` / `structure` / `cleanup` / `all`。

## 執行流程

### Step 1：計算審查範圍

- **指定路徑** → 對該路徑下的 `.js/.jsx/.ts/.tsx` 檔案

- **未指定** → 取 git 變更：
  
  ```bash
  git diff main...HEAD --name-only
  git diff --name-only
  git diff --cached --name-only
  ```
  
  彙整變更檔案，再用 `git diff -U0` 取每個檔案的行號 hunks（讓 sub-agent 聚焦在變更行而非整檔）

### Step 2：蒐集「不要質疑」脈絡

這一步是降低 false positive 的關鍵。組裝以下材料：

1. **OpenSpec 設計脈絡**（若有）：
   - 偵測 `openspec/changes/<name>/` 是否有對應本次變更的活躍 change
   - 若有，讀 `proposal.md` 與 `design.md`（若存在），萃取 1–3 句「設計決策摘要」
2. **專案規範摘要**：
   - 從 `modules/CLAUDE.md` 與相關子模組 CLAUDE.md 萃取本次 diff 觸及目錄的 sync 規則、「不同步」清單
3. **本次對話的決策**：
   - 若主 session 對話中已明確討論並決定不做某事（如「不拆 component」），列出來

若全無內容，傳「無特殊脈絡」即可，**不要**捏造。

### Step 3：讀取 rubric

讀同目錄的 `REVIEW_RUBRIC.md` 全文，準備併入 sub-agent prompt。

### Step 4：Spawn 審查 sub-agent

呼叫 Agent，參數：

- `subagent_type: "Explore"`（read-only，最適合審查）
- `description: "Code review {功能名稱}"`
- `prompt:` 依下方範本

**Prompt 範本**：

```
你是 code reviewer。針對以下變更給出獨立第二意見。

## 審查範圍
focus: {focus 值}
變更檔案與行號 hunks：
- {file1.js}（行 12-45, 88-92）
- {file2.js}（行 1-30）

## 設計脈絡（已決策事項，不要質疑）
{Step 2 蒐集的內容；若無，寫「無特殊脈絡」}

## Rubric
{REVIEW_RUBRIC.md 全文}

## 約束
1. 只審查上述變更行範圍內的程式碼
2. 只報告 Rubric 中明確列出的問題類型，不自由發揮
3. 不質疑「設計脈絡」段提及的事項
4. 每個問題必須附明確的「檔案:行號」與 Rubric 編號
5. 不建議大規模重構

## 輸出格式

### 🔴 違反 Rubric（必須處理）
- [檔案:行號] {規則編號} - 描述

### 🟡 違反 Rubric 但情境可能合理（請使用者裁決）
- [檔案:行號] {規則編號} - 描述

### 📝 TODO/FIXME 清單
- [檔案:行號] 註解內容

回報限 600 字。無發現則明確說「無 Rubric 違反」。
```

### Step 5：主 session 過濾報告

收到 sub-agent 回報後，依以下規則過濾：

| 過濾規則                                               | 動作  |
| -------------------------------------------------- | --- |
| 報告位置不在本次 diff 行號範圍內                                | 丟   |
| 與「設計脈絡」明確衝突（質疑已決策事項）                               | 丟   |
| 違反 CLAUDE.md 規範（如指責下游客製專案的專屬目錄、共用 lib 的同步邏輯） | 丟   |
| 無明確「檔案:行號」標註                                       | 丟   |
| 無對應 Rubric 編號                                      | 丟   |
| 其餘                                                 | 保留  |

若過濾後 🔴 為空、🟡 也為空，告知「無需修復」即可結束。

### Step 6：呈現裁決

過濾後的報告呈現給使用者：

- 🔴 預設採納，使用者反對才跳過
- 🟡 逐項詢問
- 📝 純資訊

### Step 7：套用修復

對使用者確認的項目，**主 session 直接用 Edit 修改**。不再呼叫 sub-agent — 修復必須在主 context 內，才能配合後續 commit / `/finalize` 流程。

若修復數量多（>5 項）或包含結構性改動，提醒使用者「結構性修改建議拆 commit」。

## 與 react-perf-check 的分工

| 檢查項目                            | react-perf-check | code-quality |
| ------------------------------- | ---------------- | ------------ |
| useEffect 依賴 / re-render / 循環依賴 | ✅                | ❌            |
| 簡化 / 命名 / 結構硬指標 / 清理            | ❌                | ✅            |

## 注意事項

- Rubric 故意收緊到「可機械驗證」項目，避免 sub-agent 自由發揮
- 結構性問題只報告硬指標（行數、props 數），不靠 sub-agent 判斷「是否該拆」
- 若 sub-agent 明顯偏離焦點，主 session 在 Step 5 過濾，不要原樣呈現給使用者
- 命名修改前確認無外部引用（grep 全 modules）
