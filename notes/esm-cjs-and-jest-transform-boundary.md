---
title: "ESM vs CommonJS：jest 為何讀不懂 node_modules 裡的套件"
type: troubleshooting
date: 2026-08-11
tags: [jest, esm, commonjs, cra, testing, module-system]
related:
  - "[React 前端測試快速上手](react-frontend-testing-quickstart.md)"
  - "[CLI 測試執行的兩個陷阱](cli-java-test-execution-pitfalls.md)"
---

> 版本：CRA 5.0.0（react-scripts）+ Jest 27.5.1 + d3 7.3.0 ｜ 案發現場：本專案 web

`npm start` 跑得好好的，`npm test` 卻在載入階段就爆 `SyntaxError`。原因不在你的程式碼，而在**兩套模組系統的過渡期**，以及**「誰負責讓 ESM 能被執行」在 dev server 與測試環境是不同的人**。

**這不是 d3 特有的問題。** 近年 chalk、node-fetch、nanoid 等大量熱門套件都已轉為 ESM-only，這是生態系的單向遷移，你會再遇到。

---

## TL;DR — 遇到時的前三步

1. **看錯誤路徑落在哪。** 落在 `node_modules/` 而非你的原始碼 → 是本文的問題。
2. **從錯誤訊息底部往上讀**，找出**你的哪個檔案**起頭了這條 import 鏈（jest 會標 `at Object.<anonymous> (src/…)`）。
3. **決定救不救這個測試**：不值得救 → 刪掉它（見解法 A）；值得救 → 覆寫 `transformIgnorePatterns`（見解法 B，附實測過的設定）。

---

## 問題描述

在 web 跑 `App.test.js`：

```
Jest failed to parse a file. ...

/…/web/node_modules/d3/src/index.js:1
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){export * from "d3-array";
                                                                                  ^^^^^^

SyntaxError: Unexpected token 'export'

> 1 | import * as d3 from 'd3';
    | ^
    at Object.<anonymous> (src/lib/shared-lib/core/palette.js:1:1)

Test Suites: 1 failed, 1 total
Tests:       0 total
```

### 兩種措辭，同一個病

錯誤訊息會依套件寫法出現**兩種不同措辭**，別把它們當成兩個問題：

| 訊息 | 出現時機 |
|---|---|
| `SyntaxError: Unexpected token 'export'` | 套件檔案以 `export` 開頭（如 d3 的 re-export 匯總檔） |
| `SyntaxError: Cannot use import statement outside a module` | 套件檔案以 `import` 開頭（如 delaunator） |

兩者都是「CJS 解析器讀到 ESM 語法」，排查方式完全相同。

---

## 判斷方式

### 兩套互不相容的模組系統

JavaScript 語言本身直到 2015 年才有模組。在那之前 Node.js 社群自己發明了 **CommonJS（CJS）**：

```js
const d3 = require('d3');      // 引入
module.exports = palette;      // 匯出
```

本質是「執行期呼叫一個叫 `require` 的函式」，所以是**動態**的 —— 可以 `require(someVariable)`，可以寫在 `if` 裡面。

ES6 才把模組寫進語言規格，就是 **ESM（ECMAScript Modules）**：

```js
import * as d3 from 'd3';      // 引入
export default palette;        // 匯出
```

`import` / `export` 是**語法結構**而非函式呼叫，必須寫在檔案最上層，引擎在**解析階段**就要能靜態分析出相依關係。

關鍵在於：**同一個 `.js` 檔不能同時被當成兩者解讀。**

> ⚠️ **常見的錯誤解釋**：「CJS 環境不認得 `export` 這個字」。不對 —— `export` 從 ES5 起就是**保留字**，`const export = 1` 在任何 JS 檔案裡都是語法錯誤。真正的原因是規格定義了**兩套文法**：Script grammar 與 Module grammar，而 `export` / `import` 宣告**只存在於 Module grammar**。用 Script 文法去解析一份 ESM 檔案，那些宣告當然不合法。差別在「用哪本文法書解析」，不在「認不認得這個字」。

### 為何 `npm start` 沒事、`npm test` 出事

因為讓 ESM 能跑起來的是**不同人**：

```
npm start  →  webpack（CRA 內建）  →  瀏覽器
              ↑ webpack 自己就是模組解析器，原生理解 import/export，
                根本不需要把它轉成 CJS

npm test   →  jest  →  Node.js（CommonJS 模式）
              ↑ 只有「你的原始碼」會先過 babel-jest 轉成 CJS
                node_modules 預設原封不動交給 Node
```

注意論點的精確形狀：**不是「webpack 有轉譯所以沒事」**（CRA 的 webpack 對 node_modules 也不做完整轉譯），而是**webpack 不需要轉譯就看得懂 ESM**。jest 則必須先把一切變成 CJS 才交給 Node。

### 那行怪東西是什麼

```js
({"Object.<anonymous>":function(module,exports,require,__dirname,__filename,jest){export * from "d3-array";
```

拆兩層看：

- **函式包裝**：這**是 Node 自己的機制**，不是 jest 發明的。Node 載入任何 CJS 模組時，都會把檔案內容包進 `function (exports, require, module, __filename, __dirname) { … }` —— 這正是 `module`、`exports`、`require` 這幾個「魔法變數」的真身：**它們一直都只是函式參數**。jest 在模仿這個包裝，並多塞了第六個參數 `jest`，好讓測試檔能直接用 `jest.mock()`。
- **`{"Object.<anonymous>": function…}` 這層物件字面量**：用物件的屬性名去替匿名函式命名，這樣 stack trace 才顯示得出名字，純粹是為了除錯體驗。

Node 解析這個函式，第一個 token 就是 `export`，爆掉。**連執行都還沒開始**，所以 `Tests: 0`。

### 為何 jest 不順手把套件也轉譯掉

jest 有轉譯能力（`babel-jest`），但劃了一條界線。CRA 5 的實際設定（`react-scripts/scripts/utils/createJestConfig.js`）：

```js
transformIgnorePatterns: [
  '[/\\\\]node_modules[/\\\\].+\\.(js|jsx|mjs|cjs|ts|tsx)$',
  '^.+\\.module\\.(css|sass|scss)$',
],
```

**node_modules 裡的 JS 一律不轉譯。** 理由是效能（node_modules 動輒數萬檔案，全過 babel 會慢到不能用）加上一個**曾經成立的假設**：那個年代 npm 上發布的套件幾乎都預先編譯成 CJS 了 —— 作者的 source 可以是 ESM 或 TypeScript，但 `npm publish` 出去的是編譯後的 CJS。

打破假設的是套件本身。看 `node_modules/d3/package.json`：

```json
{
  "version": "7.3.0",
  "type": "module",              ← 宣告：本套件的 .js 全是 ESM
  "main": "src/index.js",        ← CJS 慣例的進入點，卻指向 ESM 原始碼
  "exports": {
    "umd": "./dist/d3.min.js",   ← 只有主動宣告 umd 條件才拿得到非 ESM 版本
    "default": "./src/index.js"
  }
}
```

d3 從 v7 起**只發布 ESM**。連 `main`（傳統上該指向 CJS）都指向 ESM 原始碼。至於那份 UMD：`exports` 裡的 key 是**條件（conditions）**，由解析方宣告自己要哪一種。jest 在 CJS 模式下宣告的條件是 `require` 與 `default`，d3 沒有提供 `require`，於是落到 `default` → ESM 原始碼。`umd` 那份沒有任何人會宣告，等於不存在。

**三個條件湊齊才會炸**：

| 條件 | 檢查方式 |
|---|---|
| 套件是 ESM-only | 見下方「判斷是不是 ESM-only」 |
| 測試跑在 CJS 環境 | CRA 5 的 jest 未啟用 ESM 模式 |
| jest 不轉譯 node_modules | 專案 `package.json` 沒覆寫 `transformIgnorePatterns` |

---

## 排查步驟

### 1. 從錯誤訊息底部往上讀 import 鏈

jest 會標出**你的哪個檔案**觸發了這條鏈：

```
at Object.<anonymous> (src/lib/shared-lib/core/palette.js:1:1)
```

以 web 為例，完整鏈是：

```
App.js → AppRouter.js → 全站 applet 與 component
                      → lib/shared-lib/component/Box、Paper …
                      → lib/shared-lib/core/palette.js
                      → d3   💥
```

`import App from './App'` 這一行會**遞迴載入整棵相依樹** —— import 是有副作用的，不是只拿一個符號。

> **關於 `Tests: 0 total`**：它證明的是「失敗在工具鏈層級、不必去讀斷言」，但**不能單獨用來斷定死在 parse 階段**。模組頂層執行期丟例外、測試檔裡沒有任何 `test()`、`testMatch` 沒對到檔案，都會得到 0。指向本文問題的真正證據是 **`SyntaxError` + 路徑落在 `node_modules`** 這個組合。

### 2. 判斷是不是 ESM-only

```bash
node -e "const p=require('./node_modules/<pkg>/package.json'); \
  console.log(JSON.stringify({type:p.type, main:p.main, exports:p.exports}, null, 2))"
```

判讀時**別只看 `"type": "module"`** —— 它兩個方向都不可靠：

- 有些套件沒有 `type: module`，但改用 `.mjs` 副檔名，一樣是 ESM-only（**漏判**）。
- 有些套件有 `type: module`，卻透過 `exports` 的 `require` 條件同時提供 CJS 版本，其實不會出事（**誤判**）。

**可靠的判準**：`exports` 裡**有沒有 `require` 條件**（有就代表作者留了 CJS 出口）。最直接的實測是：

```bash
node -e "require('./node_modules/<pkg>')"   # 爆 SyntaxError 就是 ESM-only
```

### 3. 解釋「為何只有部分測試中槍」

答案是 import 鏈的長度。`util/contractTerm.test.js` 這類只 import 一個純函式檔，鏈很短，永遠碰不到 palette，所以安然無事。

反過來，`applet/TaskManagement/__tests__/comboboxIntegration.test.js` 有這一行：

```js
jest.mock('../../../lib/shared-lib/core/palette', () => ({ … }))
```

它會碰到 palette，所以寫測試的人被迫先把整個 palette 換成假物件。

> **`jest.mock` 為什麼擋得住？** babel-jest 會把 `jest.mock(…)` 呼叫**提升（hoist）到所有 `import` 之上**執行，所以真實的 palette 模組**從頭到尾沒被載入**，那條通往 d3 的鏈根本沒接通。這也是為什麼 `jest.mock` 寫在 import 後面仍然有效 —— 位置只是視覺上的，實際執行順序被 babel 改寫過了。

看到這種針對「看似無關的底層模組」的 mock，就該警覺底下埋著沒被根治的問題（見下方對解法 C 的討論）。

---

## 解決方式

### A. 刪掉那個測試

當它本來就沒價值時，這是唯一合理解。**CRA 樣板產生的 `App.test.js` 就屬於這類**：它斷言 `/learn react/i`，一旦你改掉預設頁面就永遠失敗；`render(<App />)` 還會把 env init、i18n 載入、路由全部拉起來，本來就不是能無腦 render 的東西。

> web 於 2026-08-11 刪除該檔，刪除後全套測試 18 suites / 271 tests 全綠 —— 確認它是唯一撞到這條 import 鏈的測試。

### B. 讓 jest 轉譯該套件

CRA 5 把 jest 設定鎖死，**只能寫在 `package.json` 的 `jest` 欄位**，且只有白名單內的 key 會被採用（`createJestConfig.js` 的 `supportedKeys`，`transformIgnorePatterns` 在內，寫其他 key 會被明確報錯）：

```json
{
  "jest": {
    "transformIgnorePatterns": [
      "[/\\\\]node_modules[/\\\\](?!(d3|d3-.*|internmap|delaunator|robust-predicates)[/\\\\]).+\\.(js|jsx|mjs|cjs|ts|tsx)$",
      "^.+\\.module\\.(css|sass|scss)$"
    ]
  }
}
```

**這份設定已在 web 實測通過**（2026-08-11，以一個載入 `palette.js` 的探針測試驗證：不設定則 `Tests: 0` 失敗，設定後通過）。

三個容易寫錯的點：

1. **保留 `[/\\\\]` 而不是硬寫 `node_modules/`。** 這個字元類同時涵蓋 `/` 與 `\`，硬寫斜線在 Windows 上不會匹配。網路上大量範例寫成 `node_modules/(?!…)`，在 macOS/Linux 能跑、到 Windows 同事的機器上就失效。
2. **保留結尾的副檔名錨點** `\\.(js|jsx|mjs|cjs|ts|tsx)$`，這是 CRA 原設定就有的。
3. **覆寫是整條取代**，別忘了把第二條 CSS module 的規則補回去。

`(?!…)` 是**否定前瞻**，而這個欄位的極性是反直覺的：它列的是「**要略過轉譯**的路徑」，所以把套件名加進前瞻，反而是讓它**被**轉譯。

**清單怎麼決定** —— 這是個迭代過程，不是查表：

```
補 d3 → 重跑 → 換 delaunator 爆 → 補進去 → 重跑 → …直到綠燈
```

實測佐證：只寫 `(?!(d3|d3-.*)/)` 時，下一個爆的是 `delaunator`（d3 的**間接**相依，訊息還換成 `Cannot use import statement outside a module`）。所以**不要照抄任何文章的清單**，包括這一份 —— 你的相依樹跟這裡的不同。可以照抄的是**形狀**（字元類、否定前瞻、副檔名錨點、記得補回 CSS 那條）。

> 用 `d3-.*` 這種萬用字元是可以的，代價只是順帶轉譯了一些本來就是 CJS 的 `d3-*` 套件（頂層 `node_modules` 被 hoist 的版本不一定與 d3 直接依賴的相同），多花一點轉譯時間而已，不會出錯。真正無法概括的是 `internmap`、`delaunator` 這類**名字跟主套件毫無關係的間接相依** —— 只能靠重跑一個一個撞出來。

### C. mock 掉中間層

`jest.mock` 掉 palette 這種轉接檔。**這個做法的定位取決於你在測什麼**：

- **合理**：被測元件根本不關心配色，palette 只是它相依樹上的過路財神。這時 mock 掉是正常的測試隔離，不是欠債。
- **欠債**：被測的正是配色行為本身，mock 掉等於把要驗證的東西換成假的，測試就失去意義了。

判別方式很簡單：**如果 mock 的回傳值隨便填都不影響斷言成敗，那就是前者。** web 的 `comboboxIntegration.test.js` 屬於前者，所以它是合理繞道；但它同時也是一塊化石 —— 它記錄了「有人撞過這個問題，用局部方案繞開了，沒有人回頭處理根因」。

### 怎麼選

A 和 B **不是互斥選項，而是時序**：現在刪掉不值得救的測試（A），等到真的要為 StyledChips、FunnelChart 這類實際用到 palette 的元件寫測試時，再付 B 的成本。**不用預先做。** C 是在 A/B 之外的正交手段，看上面的判別方式決定它算隔離還是算欠債。

### 其他選項與為何不選

- **降版 d3 到 v6**（最後的 CJS 版本）：把生態系往回拉，且 v6 不再修 bug。
- **啟用 jest ESM 模式**（`NODE_OPTIONS=--experimental-vm-modules`）：jest 27 的 ESM 支援仍標為 experimental，主要缺口在 mock —— ESM 的模組繫結是靜態的，不能像 CJS 那樣事後改寫模組表，所以 `jest.mock` 無法運作，要全面改寫成 `jest.unstable_mockModule`（名字裡的 `unstable_` 就是 API 未定案的自白）。會波及所有既有測試。
- **改用 Vite + Vitest**：Vitest 原生跑 ESM，這個問題整類消失。但那是換建置工具的大工程，不是修一個測試的手段。

---

## 教訓

1. **`Tests: 0 total` 與「測試失敗」是兩件不同的事。** 前者代表工具鏈壞了，後者才是程式碼的問題。看到 0 就別再讀斷言 —— 但也別只憑 0 就斷定是本文的問題，要配合 `SyntaxError` + `node_modules` 路徑一起判讀。
2. **測試環境與 dev server 的模組解析是兩套獨立系統。** 「網頁跑得起來」不保證「測試載得進來」，反之亦然。
3. **`jest.mock` 一個看似無關的底層模組，往往是舊傷的疤。** 檢視既有測試的 mock 清單，能反推出哪些相依鏈是有毒的 —— 即使那個 mock 本身是合理的隔離。
4. **這類問題會再發生。** 生態系正在單向遷移到 ESM-only，而 CRA 5 已停止維護、jest 27 的 CJS 假設不會改。每次新增畫圖表、色彩、日期處理這類現代套件，都可能再撞一次。真正的長期解是換掉建置工具，不是把 `transformIgnorePatterns` 的清單養得越來越長。

---

## 相關

- [React 前端測試快速上手](react-frontend-testing-quickstart.md) — 同一組工具棧（CRA 5 + Jest 27 + RTL 12）的實務寫法；本篇是它的「跑不起來時」對照
- [CLI 測試執行的兩個陷阱](cli-java-test-execution-pitfalls.md) — 後端側的同型問題：測試失敗的原因不在程式碼而在工具鏈預設值。該篇的「綠燈不等於有跑」與本篇的「`Tests: 0` 不是測試失敗」是同一個判讀習慣的兩面

## 相關檔案

- `web/src/lib/shared-lib/core/palette.js` — 引入 d3 的源頭
- `web/src/applet/TaskManagement/__tests__/comboboxIntegration.test.js` — 用 `jest.mock` 繞開 palette 的既有案例
- `web/node_modules/react-scripts/scripts/utils/createJestConfig.js` — CRA 的 jest 設定與 `supportedKeys` 白名單
