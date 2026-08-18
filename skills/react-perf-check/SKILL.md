---
name: react-perf-check
description: 檢查 React 元件的效能問題（useEffect 依賴、Context value 穩定性、循環依賴、memo 使用錯誤），由獨立 sub-agent 執行以避免「自己 review 自己」的盲點。Use when 前端功能完成後的效能審查、API 重複調用排查、或被 /finalize 串接呼叫。
argument-hint: "[檔案路徑或目錄，省略則取 git diff] [focus=effect-deps|context|jsx-inline|memoization|all]"
---

# react-perf-check

對 **$ARGUMENTS** 進行 React 效能問題檢查。

**核心策略**：由獨立 sub-agent 在乾淨 context 中執行檢查，主 session 只負責蒐集脈絡、過濾報告、套用修復。避免「寫 code 的人 = review 的人」造成的合理化偏誤。

## 適用場景

- 前端功能開發完成後的效能審查
- 發現 API 重複調用時的問題排查
- 被 `/finalize` skill 串接呼叫（僅前端改動時）

## focus 參數

預設 `all`：`effect-deps` / `context` / `jsx-inline` / `memoization` / `all`。

## 執行流程

### Step 1：計算審查範圍

- **指定路徑** → 對該路徑下的 `.js/.jsx/.ts/.tsx`
- **未指定** → 取 git 變更檔案，過濾出含以下任一者：
  - React hook（`useEffect` / `useState` / `useMemo` / `useCallback` / `useContext`）
  - JSX（`.jsx` / `.tsx` 副檔名，或 `.js` 內含 JSX）
  
  非前端 / 純後端檔案直接跳過。

用 `git diff -U0` 取每個檔案的變更行號 hunks。

### Step 2：蒐集「不要質疑」脈絡

組裝以下材料給 sub-agent：

1. **OpenSpec 設計脈絡**（若有）：
   - 若 `openspec/changes/<name>/design.md` 提及「刻意 re-render 設計」「不 memo 的原因」等決策，列出
2. **已知穩定的自訂 hook 清單**：
   - 從各前端模組的 `src/hook/` 掃描，列出已用 useMemo/useCallback 包裹返回值的 hook 名稱（避免 sub-agent 誤判為不穩定）
3. **本次對話的決策**：
   - 若主 session 已明確討論並決定不做某 perf 優化，列出

若全無內容，傳「無特殊脈絡」即可，不要捏造。

### Step 3：讀取 rubric

讀同目錄的 `PERF_RUBRIC.md` 全文。

### Step 4：Spawn 審查 sub-agent

呼叫 Agent：
- `subagent_type: "Explore"`（read-only；可讀 hook 定義以判斷穩定性）
- `description: "Perf check {功能名稱}"`

**Prompt 範本**：

```
你是 React 效能 reviewer。針對以下變更找出**確定會造成 bug 或重複 render** 的問題，**不做主觀優化猜測**。

## 審查範圍
focus: {focus 值}
變更檔案與行號 hunks：
- {file1.jsx}（行 12-45）
- {file2.js}（行 80-110）

## 設計脈絡（已決策，不要質疑）
{Step 2 蒐集內容；若無，寫「無特殊脈絡」}

## 已知穩定的自訂 hook（不要列為不穩定）
{Step 2 蒐集的 hook 清單；若無，寫「無資訊」}

## Rubric
{PERF_RUBRIC.md 全文}

## 約束
1. 只審查上述變更行的程式碼
2. 只報告 Rubric 中列出的具體模式，不自由發揮
3. **禁止泛泛建議「考慮使用 memo/useMemo/useCallback」**
4. 不質疑「設計脈絡」段提及的事項
5. 每個問題必須附明確「檔案:行號」與 Rubric 編號
6. 若需要查 hook 來源以判斷穩定性，主動 Read 該 hook 定義檔

## 輸出格式

### 🔴 確定問題（會造成 bug 或重複 API 呼叫）
- [檔案:行號] {規則編號} - 描述

### 🟡 確定模式違反但效能影響待評估（請使用者裁決）
- [檔案:行號] {規則編號} - 描述

回報限 600 字。無發現則明確說「無 Rubric 違反」。
```

### Step 5：主 session 過濾報告

依以下規則過濾：

| 過濾規則 | 動作 |
|---------|------|
| 報告位置不在 diff 行號範圍內 | 丟 |
| 屬於 PERF_RUBRIC.md 第 5 段「不報告」清單 | 丟 |
| 無明確「檔案:行號」標註 | 丟 |
| 無對應 Rubric 編號 | 丟 |
| 「可能會 re-render」這類無實證描述 | 丟 |
| 與「設計脈絡」明確衝突 | 丟 |
| 其餘 | 保留 |

若過濾後 🔴 / 🟡 都為空，告知「無需修復」即可結束。

### Step 6：呈現裁決

- 🔴 預設採納（這是確定 bug）
- 🟡 逐項詢問
- 修復常牽涉跨檔重構（加 useMemo 包 Provider value、移 selector 到外部），呈現時要明確標示影響範圍

### Step 7：套用修復

主 session 用 Edit 套用使用者確認的項目。**注意**：
- 修復常涉及多檔 → 提醒「拆成獨立 commit」
- 加 `useMemo` / `useCallback` 後要確認對應 import 也跟著加
- 移 selector 到外部時要確認沒有 closure 依賴外部變數
- `eslint-disable-next-line react-hooks/exhaustive-deps` 只在主 session 確認安全時使用

## 與 code-quality 的分工

| 檢查項目 | code-quality | react-perf-check |
|----------|--------------|------------------|
| 簡化 / 命名 / 清理 | ✅ | ❌ |
| 結構硬指標（行數、props 數） | ✅ | ❌ |
| useEffect 依賴 / 循環依賴 | ❌ | ✅ |
| Context value 穩定性 | ❌ | ✅ |
| memo / useMemo / useCallback 使用錯誤 | ❌ | ✅ |

## 注意事項

- Rubric 故意收緊到「確定問題」，避免過度優化建議
- `React.memo` / `useMemo` / `useCallback` 都不是免費的 — sub-agent 不得無實證建議加上
- 修復常涉及多檔 → 拆 commit
- 移除依賴前要確認該值在 effect 邏輯中不需要響應變化
