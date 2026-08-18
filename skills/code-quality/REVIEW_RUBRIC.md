# Code Review Rubric

供 code-quality skill 的 sub-agent 使用。**每一項都必須可以二元判定**（是/否有此問題），不做開放性「考慮…」「可優化…」的建議。

報告時請對應到 Rubric 編號（如 `1.1`、`4.3`）。

---

## 1. 簡化（focus=simplify）

### 1.1 函數過長

- **規則**：單一函數超過 50 行（含空白行與註解）
- **動作**：報告檔案、行號、函數名稱、實際行數

### 1.2 巢狀過深

- **規則**：`if` / `for` / `while` 巢狀超過 3 層
- **動作**：報告最內層位置與當前深度

### 1.3 重複的程式碼區塊

- **規則**：同一檔案內，連續 5 行以上幾乎相同的程式碼出現 2 次以上
- **不報告**：抽象的「看起來像可以共用」、跨檔案的相似性

### 1.4 可機械化簡寫

| 原本                                       | 應簡化為                 |
| ---------------------------------------- | -------------------- |
| `if (x === true)`                        | `if (x)`             |
| `if (x === false)`                       | `if (!x)`            |
| `x ? true : false`                       | `!!x` 或 `Boolean(x)` |
| `if (x) return true; else return false;` | `return x;`          |
| `if (x) return true; return false;`      | `return !!x;`        |

只報告以上列出的具體形式。

### 1.5 過度複雜的條件

- **規則**：單一 `if` 條件含 3 個以上 `&&` 或 `||`
- **動作**：建議拆出有意義的中間變數

---

## 2. 命名（focus=naming）

### 2.1 命名慣例違反

| 類型                 | 慣例                     | 違反範例                           |
| ------------------ | ---------------------- | ------------------------------ |
| 元件                 | PascalCase             | `userProfile` / `user_profile` |
| 函數 / 變數            | camelCase              | `Handle_click` / `HandleClick` |
| 常數（module 層級的字面值）  | UPPER_SNAKE_CASE       | `maxRetryCount = 3`            |
| Boolean 變數 / props | `is/has/should/can` 前綴 | `loading`（應為 `isLoading`）      |
| 事件 handler         | `handle` 前綴            | `onClickButton`（內部 handler）    |
| Callback prop      | `on` 前綴                | `handleClick`（傳給子元件的 prop）     |

### 2.2 Magic numbers / strings

- **規則 a**：非 `0` / `1` / `-1` 的數字字面值出現在條件判斷、迴圈邊界、timeout/delay 設定 → 報告
- **規則 b**：條件判斷或 switch case 中比對的字串字面值（status code、role、type） → 報告
- **不報告**：陣列 index、CSS 數值（除非用於業務邏輯）

### 2.3 禁用的模糊命名

只在以下精確情況報告：

- 變數名為 `data` / `info` / `item` / `obj` / `result` 且非 callback 參數
- 變數名為 `temp` / `tmp`
- 單字母變數 `x` / `y` / `i` 且非迴圈索引或座標
- 流水號命名 `handleClick1` / `handleClick2` / `fn2` / `helper3`

**不報告**：「命名是否更具表達力」這類主觀判斷。

---

## 3. 結構（focus=structure）

僅報告硬指標，**不**做主觀建議。

### 3.1 元件過大

- **規則**：單一元件檔案超過 300 行（含 JSX）
- **動作**：報告檔名與實際行數

### 3.2 Props 過多

- **規則**：單一 React 元件接收 props 超過 7 個
- **動作**：報告元件名與 props 數量

### 3.3 useState + useEffect 重複模式

- **規則**：同一檔案內出現 2 次以上「`useState` for data + `useState` for loading + `useEffect` for fetch」的精確組合
- **動作**：標註可抽 custom hook 的位置

**不報告**：

- 「考慮抽出 custom hook」（除非命中 3.3）
- 「關注點是否分離」
- 「容器/展示元件是否拆分」

---

## 4. 清理（focus=cleanup）

最容易機械驗證的類別。

### 4.1 未使用的 import

- **規則**：`import { A, B }` 中 `A` 或 `B` 在檔案內無任何引用 → 報告
- **包含**：未使用的整個 import 語句

### 4.2 未使用的變數

- **規則**：宣告後從未讀取的變數（不含 `_` 開頭的 placeholder）

### 4.3 console 殘留

- **規則**：`console.log` / `console.debug` / `console.warn` → 報告
- **不報告**：`console.error`（錯誤日誌可保留）

### 4.4 註解掉的程式碼

- **規則**：超過 3 行的連續註解（內容明顯是程式碼，非說明文字）

### 4.5 Dead code

- **規則 a**：`return` / `throw` 之後同一 block 還有可執行程式碼
- **規則 b**：`if (false)` / `if (true)` 字面值條件
- **規則 c**：永不滿足的條件（如先 return 已排除的值，後又判斷）

---

## 5. TODO / FIXME 蒐集

列出所有：

- `TODO`
- `FIXME`
- `HACK`
- `XXX`

**這不是違反**，是純資訊呈現。放在報告的 📝 段落。

---

## 報告格式

對每個發現的問題，使用以下格式：

```
[檔案:行號] 規則編號 - 一句話描述
```

範例：

```
CollapsiblePipelineHeader.jsx:88   2.2  magic number 300
useCollapse.js:12                  1.1  函數長 67 行（超過 50 行）
PipelineList.jsx:45                4.3  console.log 殘留
```

## 嚴格禁止事項

Sub-agent 不得：

1. 給出「可以考慮…」「建議評估…」「或許可以…」開放性建議
2. 質疑既定的架構決策（見 prompt 的「設計脈絡」段）
3. 評論非 diff 行號範圍內的程式碼
4. 報告 Rubric 未列出的問題類型
5. 評論程式碼風格偏好（縮排、引號、空白行）— 那是 linter 的工作
6. 對「做得好的地方」給泛泛讚揚（如「結構清晰」「命名良好」）— 若要肯定，必須指明具體行號與具體做法
