# React 前端測試快速上手

> 版本：CRA 5 + React 17 + Jest 27 + React Testing Library 12 | 專案：本專案 web / portal

寫給「Java 後端測試熟悉、但沒寫過 React 測試」的人。聚焦 custom hook 與 component 兩種最常見場景。

---

## 工具棧（已內建，不需另外安裝）

| 工具 | 角色 | Java 對照 |
|---|---|---|
| **Jest** | Test runner + assertion + mock | JUnit + Mockito |
| **React Testing Library** | 渲染 component、查詢 DOM | （無對照，這是 React 特有） |
| **@testing-library/jest-dom** | 加 DOM 專屬 matcher（`toBeInTheDocument` 等） | AssertJ 風格擴充 |
| **@testing-library/user-event** | 模擬使用者操作（點擊、輸入） | Selenium-like |

`web/package.json` 已含全部，CRA 預設帶。

---

## 跑測試

```bash
cd web
npm test                         # watch mode（檔案變動自動重跑）
npm test -- --watchAll=false     # 跑一次就退出（CI 用）
npm test -- --coverage           # 含 coverage report
npm test -- useFlowGraph         # 只跑檔名含 "useFlowGraph" 的測試
```

`react-scripts test` 會自動找 `*.test.js` / `*.spec.js` / `__tests__/` 底下的檔案。

---

## 檔案位置慣例

兩種放法，CRA 都支援，**選一種一致即可**：

```
src/component/ApprovalFlowEditor/
├── hooks/
│   ├── useFlowGraph.js
│   └── useFlowGraph.test.js          ← 共置（推薦）
└── ...

或

src/component/ApprovalFlowEditor/
├── hooks/
│   └── useFlowGraph.js
└── __tests__/
    └── useFlowGraph.test.js          ← 集中
```

**推薦共置**（與被測檔案放一起）：找測試容易、refactor 移動檔案不會漏。

---

## 範例 1：測 Custom Hook（最常見）

`useFlowGraph` 是個典型 custom hook — 回傳 state 與 operations。測法用 React Testing Library 的 `renderHook` + `act`。

> ⚠️ **本專案使用 @testing-library/react v12，沒有 `renderHook`**（v13+ 才內建）。
> 解法：在測試檔頂部寫一個 wrapper component shim，不需新增依賴：
>
> ```js
> import React from 'react';
> import {render, act} from '@testing-library/react';
>
> function renderHook(hookFn) {
>     const ref = {current: null};
>     function Wrapper() {
>         ref.current = hookFn();
>         return null;
>     }
>     render(<Wrapper/>);
>     return {result: ref};
> }
> ```
>
> 之後的 `result.current.xxx` 用法與 v13+ 完全一致。

```js
// useFlowGraph.test.js
import { renderHook, act } from '@testing-library/react';
import useFlowGraph from './useFlowGraph';

describe('useFlowGraph', () => {

    describe('addStandaloneNode', () => {
        it('在空 graph 加一個 approval 節點 → nodes 長度變 1', () => {
            const { result } = renderHook(() => useFlowGraph({ nodes: [], edges: [] }));

            act(() => {
                result.current.addStandaloneNode('approval');
            });

            expect(result.current.nodes).toHaveLength(1);
            expect(result.current.nodes[0].nodeType).toBe('approval');
            expect(result.current.edges).toHaveLength(0);
        });
    });

    describe('removeNode', () => {
        it('刪除 1 入 1 出的中間 approval → 上下游自動 join', () => {
            const initial = {
                nodes: [
                    { uuid: 'A', nodeType: 'approval' },
                    { uuid: 'P', nodeType: 'approval' },
                    { uuid: 'B', nodeType: 'approval' },
                ],
                edges: [
                    { uuid: 'e1', fromNodeUuid: 'A', toNodeUuid: 'P', isDefault: true },
                    { uuid: 'e2', fromNodeUuid: 'P', toNodeUuid: 'B', isDefault: true },
                ],
            };
            const { result } = renderHook(() => useFlowGraph(initial));

            act(() => {
                result.current.removeNode('P');
            });

            expect(result.current.nodes.map(n => n.uuid)).toEqual(['A', 'B']);
            expect(result.current.edges).toHaveLength(1);
            expect(result.current.edges[0].fromNodeUuid).toBe('A');
            expect(result.current.edges[0].toNodeUuid).toBe('B');
        });

        it('刪除 condition 節點 → 下游節點保留為孤島子圖', () => {
            // 略：見上面結構
        });
    });
});
```

### `act()` 是什麼？

包住「會改變 React state 的動作」。不包會被 React warn「state update not wrapped in act」。心智模型：**「告訴 React：我要修改 state，請等所有 effect 跑完再讓我斷言」**。

```js
// 正確
act(() => {
    result.current.addStandaloneNode('approval');
});
expect(result.current.nodes).toHaveLength(1);

// 錯誤（會 warn）
result.current.addStandaloneNode('approval');
expect(result.current.nodes).toHaveLength(1);
```

### `result.current` 是什麼？

`renderHook` 回傳 `{ result }`，`result.current` 是「**最新一次** hook 回傳的物件」。每次 state 變動後 `result.current` 自動指向新值，所以斷言時用 `result.current.xxx` 取得最新狀態。

---

## 範例 2：測 Component

```js
// AddNodePopover.test.js
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import AddNodePopover from './AddNodePopover';

describe('AddNodePopover', () => {
    it('點擊「增加簽核步驟」→ onSelect 收到 "approval"', async () => {
        const onSelect = jest.fn();
        const anchorEl = document.createElement('div');
        document.body.appendChild(anchorEl);

        render(
            <AddNodePopover
                anchorEl={anchorEl}
                onSelect={onSelect}
                onClose={() => {}}
            />
        );

        await userEvent.click(screen.getByText('增加簽核步驟'));

        expect(onSelect).toHaveBeenCalledWith('approval');
    });
});
```

### 查詢 DOM 的優先順序

React Testing Library 鼓勵「**用使用者看到的方式找元素**」：

| 優先級 | 查詢方法 | 對應 |
|---|---|---|
| 1 | `getByRole('button', { name: '...' })` | 無障礙標籤 |
| 2 | `getByLabelText('...')` | 表單 label |
| 3 | `getByPlaceholderText('...')` | input placeholder |
| 4 | `getByText('...')` | 文字內容 |
| 5 | `getByTestId('...')` | 最後手段，需在元件加 `data-testid` |

**避免** `container.querySelector('.some-class')` — 把 CSS class 當 test selector 是脆弱耦合。

---

## Mock 外部依賴

### Mock service / module

```js
jest.mock('../../service/condition/conditionFieldService', () => ({
    getHeadcountApprovalFields: jest.fn().mockResolvedValue([
        { path: 'requisition.jobTitle', type: 'STRING', operators: ['EQ', 'CONTAINS'] },
    ]),
}));
```

### Mock i18n

web 用自家 i18n（`src/lib/shared-lib/core/i18n`）。測試時通常直接 mock 成 identity 函式：

```js
jest.mock('../../../lib/shared-lib/core/i18n', () => ({
    translate: (key) => key,    // 回傳 key 本身
}));
```

---

## 應用到 ApprovalFlowEditor refactor 的最小測試集

針對 Change A，建議優先寫的 hook test cases（每個寫一個 `it`）：

```
useFlowGraph
├── addStandaloneNode
│   └── 在空 graph 加 approval → nodes 長度 1、edges 0
├── addOutgoingEdge
│   ├── 從某節點加懸掛邊 → edge 新增、toNodeUuid null
│   └── 從不存在節點加 → 無動作（or throw）
├── attachEdgeTarget
│   ├── 設定懸掛邊 toNode → edge 完成
│   └── 設定到自身 → 拒絕
├── removeNode
│   ├── 1 入 1 出 approval → 自動 join、入邊 toNode 改指下游
│   ├── condition → 下游子圖隔離（出邊刪、下游節點保留）
│   ├── sink approval → 純刪、入邊變懸掛
│   └── entry approval → 純刪、下游節點變孤島
├── convertToBranch
│   ├── approval → condition + 1 條 default 懸掛邊
│   └── condition 已存在 → 無動作
└── pasteTemplate
    ├── 套到空畫布 → 整段拼接、entry uuid 回傳
    └── 套到懸掛邊末端 → 邊 toNode 設為範本 entry
```

不必追求 100% coverage — 把**業務 invariants** 與**容易出錯的邊角**寫成測試即可。

---

## 常見坑

### 坑 1：useEffect 內的非同步沒等就斷言

```js
// 錯誤
act(() => {
    result.current.fetchData();
});
expect(result.current.data).toBeDefined();   // 可能還沒 resolve

// 正確
await act(async () => {
    await result.current.fetchData();
});
expect(result.current.data).toBeDefined();
```

### 坑 2：直接 mutate state

```js
// 錯誤（hook 內部 setState 沒被通知）
result.current.nodes.push({ uuid: 'X' });

// 正確（透過 hook 提供的 operation）
act(() => {
    result.current.addStandaloneNode('approval');
});
```

### 坑 3：閉包陷阱

```js
const { result } = renderHook(() => useFlowGraph());
const { addNode } = result.current;        // ❌ 解構後拿到的是「當下這刻」的 addNode

act(() => addNode('approval'));            // 可能拿到舊的 setState

// 正確：每次都從 result.current 取
act(() => result.current.addNode('approval'));
```

---

## 下一步

- 從一個小 utility 函式開始寫測試找手感（**別**從 CRA 預設帶的 `App.test.js` 起手 — 它斷言 `/learn react/i`，在真實專案必然失敗，而且會撞上 [ESM/CJS 與 Jest transform 邊界](esm-cjs-and-jest-transform-boundary.md) 描述的 d3 轉譯問題，連跑都跑不起來。web 已於 2026-08-11 刪除該檔）
- Custom hook 測試從 `useFlowGraph` 的 `removeNode` 切入（行為複雜、值得保護）

更深入：
- [Testing Library 官方文件](https://testing-library.com/docs/react-testing-library/intro/)
- [Kent C. Dodds 的 Testing JavaScript](https://testingjavascript.com/)（付費但業界經典）

## 相關
- [ESM/CJS 與 Jest transform 邊界](esm-cjs-and-jest-transform-boundary.md) — 同一組工具棧「測試連載入都失敗」時的排查：ESM-only 套件 vs jest 的 CommonJS 環境
