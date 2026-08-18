# React Performance Rubric

供 react-perf-check skill 的 sub-agent 使用。每一項都對應**具體的程式碼模式**，不做「考慮使用 X」的開放性建議。

報告時請對應到 Rubric 編號（如 `1.2`、`2.1`）。

---

## 重要原則

效能檢查容易誤判。本 rubric 只列**確定會造成 bug 或重複 render 的模式**，不列「可能可以優化」的猜測。

特別禁止：
- 建議所有純展示元件都加 `React.memo`（無實證問題前不建議）
- 建議所有 JSX inline arrow function 改 `useCallback`
- 建議所有 inline object 改 `useMemo`
- 「可以考慮使用 useMemo 來優化」這類無實證建議

---

## 1. useEffect 依賴問題（focus=effect-deps）

### 1.1 不穩定 hook 返回值作為依賴

**規則**：`useEffect` 的依賴陣列包含以下任一：
- `usePrompt()` 返回值
- `useTheme()` 返回值（除非確定來源穩定）
- `useSelector(inlineSelector)` 的 inline selector 結果
- 自訂 hook 返回的物件/函數，且該 hook 內部未 `useMemo` / `useCallback` 包裹

**報告**：useEffect 行號 + 依賴陣列中的問題項目 + 該 hook 的引入位置（若可見）

### 1.2 循環依賴

**規則**：useEffect 內部呼叫 `setX(...)`，且 `x` 出現在同一 useEffect 的依賴陣列中

**報告**：useEffect 行號 + setter 名稱 + 被循環依賴的 state 名稱

### 1.3 內部 inline 函數作為依賴

**規則**：依賴陣列含函數，該函數在元件函式體內以 `const fn = () => {...}` 形式定義（非 useCallback 包裹）

**不報告**：模組層級定義的函數、props 傳入的函數（除非有額外證據顯示來源不穩定）

### 1.4 解構函數作為依賴

**規則**：依賴陣列含 `const { fn } = useHook()` 解構出的函數，且 hook 內部未保證 fn 穩定

---

## 2. Context Provider value 穩定性（focus=context）

### 2.1 Provider value 為 inline 物件

**規則**：`<XxxContext.Provider value={{ ... }}>` 或 `value={[..., ...]}` 直接 inline 在 JSX 中

**報告**：Provider 行號 + Context 名稱

### 2.2 Provider value 為非 memoized 函數呼叫

**規則**：`<XxxContext.Provider value={createValue()}>`，且 `createValue` 非 `useMemo` 結果

---

## 3. JSX 中傳給 memoized 子元件的不穩定 props（focus=jsx-inline）

只在以下情境報告，避免過度警告：

### 3.1 傳給 `React.memo` 子元件的 inline 函數/物件

**規則**：父元件中 `<MemoChild prop={() => ...} />` 或 `<MemoChild prop={{ ... }} />`，且 `MemoChild` 是以 `React.memo()` 包裹的元件

**報告**：父元件行號 + 子元件名稱 + 哪個 prop

**不報告**：傳給非 memo 子元件的 inline props（memo 才是優化前提）

---

## 4. memo / useMemo / useCallback 使用錯誤（focus=memoization）

### 4.1 useMemo / useCallback 依賴陣列遺漏

**規則**：`useMemo` / `useCallback` 的工廠函數內讀取了外部變數，但該變數未列入依賴陣列

**報告**：行號 + 缺漏依賴名稱

### 4.2 useMemo 包裝原始值

**規則**：`useMemo(() => primitiveValue, [...])`，回傳值是 number / string / boolean

**理由**：原始值不需要 memo，反而增加開銷

### 4.3 useCallback 依賴包含 setState

**規則**：`useCallback` 依賴陣列含 `setX`（`useState` 的 setter，React 保證穩定）

**理由**：無害但表示對 React 規則理解偏差，可建議移除

---

## 5. 明確不報告的事項

以下是常見的過度優化建議，**絕對不要**在報告中出現：

1. 「純展示元件可加 `React.memo`」— memo 不是免費的，必須有實證
2. 「所有 inline arrow 應改 `useCallback`」— 沒傳給 memo 子元件就不必要
3. 「考慮拆出 selector」— 除非已命中 1.1
4. 「列表項目應加 key 為 X」— 那是另一個檢查範疇（屬 code-quality 範圍）
5. 效能猜測（「可能會慢」、「看起來會 re-render」）— 必須對應 1-4 的具體規則
6. 「這個 useEffect 太大應拆」— 屬 code-quality 範圍

---

## 報告格式

對每個發現的問題，使用以下格式：

```
[檔案:行號] 規則編號 - 一句話描述
```

範例：
```
PipelineProvider.js:45    1.1  useEffect 依賴含 usePrompt() 返回值
RequisitionList.jsx:128   2.1  Context Provider value 為 inline 物件
ActionMenu.jsx:67         4.2  useMemo 包裝 string 原始值
CandidateCard.jsx:23      1.2  循環依賴：setLoadedUuid 在依賴含 loadedUuid 的 effect 中呼叫
```

## 嚴格禁止事項

Sub-agent 不得：
1. 給出「可以考慮…」「建議評估…」「或許可以…」開放性建議
2. 評論非 diff 行號範圍內的程式碼
3. 報告第 5 段明列的「不報告」項目
4. 猜測「這個 hook 可能不穩定」— 必須有 Rubric 1.1 清單依據或主動 Read hook 定義確認
5. 對「做得好的地方」泛泛讚揚
6. 評論程式碼風格或結構問題（那是 code-quality 範圍）
