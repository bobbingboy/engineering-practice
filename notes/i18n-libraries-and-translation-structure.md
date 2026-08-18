---
title: i18n 函式庫選擇與翻譯結構設計
type: feature
date: 2026-05-21
tags: [i18n, frontend, library-selection, translation]
related: [React 前端測試快速上手](react-frontend-testing-quickstart.md)
---

## 概述

這篇文件回答兩個我在實際維護 i18n 時反覆遇到的痛點：

1. **單一翻譯檔不斷膨脹，想拆分** — 何時該拆、怎麼拆、拆檔的工具鏈支援
2. **英文同字、中文要不同翻譯** — 例如 `name` 在候選人是「姓名」、在職缺是「職稱」，怎麼避免撞名

回答上述問題需要先理解兩個基礎概念：**case-sensitivity**（key 大小寫敏感性）和 **namespace**（翻譯分組空間）。這兩個機制決定了任何 i18n 函式庫的撞名風險與拆檔可能性。

**範圍**：
- ✅ React 生態（i18next、react-intl、Lingui）+ Vue（vue-i18n）為主
- ✅ 跨函式庫選擇與翻譯檔結構設計
- ✅ TypeScript 型別、自動 extraction、翻譯平台等周邊議題（摘要式）
- ❌ 後端 i18n（Spring MessageSource、Django i18n 等）
- ❌ 特定框架的 i18n（Next.js、Remix 內建方案）
- ❌ 個別函式庫的歷史版本變遷

**讀者**：未來的我，重新開新專案 / 接手既有專案 / 評估該不該換 i18n 函式庫時。

## 核心概念

### case-sensitive：key 是否區分大小寫

決定 `t('Apply')` 與 `t('apply')` 是不是同一個 key。看起來是小細節，實際影響翻譯撞名風險與工具鏈支援度。

#### 主流函式庫的預設行為

| 函式庫 | 預設 case-sensitivity |
|--------|---------------------|
| i18next | ✅ Case-sensitive |
| react-intl (FormatJS) | ✅ Case-sensitive |
| Lingui | ✅ Case-sensitive |
| vue-i18n | ✅ Case-sensitive |

**所有主流函式庫的預設都是 case-sensitive**。也就是 `t('Apply')` 與 `t('apply')` 視為**不同 key**，互不干擾。

#### 為什麼有人會關掉

少數情境（多為自訂 wrapper）會強制 key lowercase：

- **避免「翻譯漏字」**：希望 `t('apply')` 和 `t('Apply')` 都能命中同一筆翻譯，減少譯者重複翻譯
- **歷史包袱**：早期專案的 key 命名沒規範，後來加上 lowercase 統一處理
- **錯誤的「容錯設計」**：以為強制 lowercase 能讓開發者「不需在意大小寫」

#### 關掉後的代價

放棄 case-sensitive 會帶來連鎖副作用：

- **撞名風險爆增**：`Apply`（應徵）與 `apply`（套用）被視為同 key，原本只是「不同大小寫的相同字」變成「會互相覆蓋的衝突」
- **與 namespace 機制衝突**：當 namespace 也被合併成單一池時（如 shared-lib），lowercase 進一步壓縮 key 空間
- **工具鏈失效**：i18next-parser、TypeScript 型別生成等工具都假設 case-sensitive，遇到強制 lowercase 的 wrapper 會行為異常
- **與主流慣例脫節**：第三方元件庫的翻譯 key（如 MUI、Chakra）通常是 PascalCase 或 camelCase，被 lowercase 後可能撞既有 key

#### 實務建議

**永遠保持 case-sensitive**。即使想做「容錯」，也應在呼叫端做 normalization（如 `t(key.toLowerCase())`），而非從根本上關閉 case-sensitivity。

若接手的專案已強制 lowercase（如 本專案 的 shared-lib wrapper），**所有命名策略要假設 case-insensitive 才安全** — 詳見「Key 命名策略」與「自寫 wrapper」段落。

### Namespace：翻譯的「分組空間」

把 namespace 想像成**不同抽屜的標籤**：同樣寫「name」的標籤紙，放在「候選人」抽屜代表「姓名」、放在「職缺」抽屜代表「職稱」。Namespace 就是這層「抽屜」，讓相同 key 在不同情境下指向不同翻譯。另一個常見類比是**部門內線**：「分機 101」在業務部找的是王經理、在工程部找的是李工程師，分機號碼相同但歸屬部門不同。

#### 沒 namespace vs 有 namespace

沒有 namespace 時，所有 key 攤平在同一個全域池：

```json
{
  "name": "姓名",
  "save": "儲存"
}
```

如果候選人功能想要 `name = "姓名"`、職缺功能想要 `name = "職稱"`，兩者必然撞名 — 後寫入的覆蓋先寫入的，全站受波及。

加上 namespace 後，key 變成「namespace + key」的組合：

```json
// candidate.json
{ "name": "姓名" }

// requisition.json
{ "name": "職稱" }
```

呼叫端用 namespace 隔離：

```js
t('candidate:name')    // → "姓名"
t('requisition:name')  // → "職稱"
```

#### 邏輯 namespace vs 實體分檔

兩個概念常被混為一談，但其實是不同維度：

| 維度 | 概念 | 對應實作 |
|------|------|---------|
| **邏輯 namespace** | 程式碼層的語意分組 | `t('candidate:name')` 中的 `candidate` |
| **實體分檔** | 檔案系統上的拆檔 | `locales/zh/candidate.json` |

通常實作是「一個 namespace = 一個檔」，但兩者可分離：
- 一個 namespace 可由多檔組成（例如 `candidate-basic.json` + `candidate-resume.json` 都掛進 `candidate` namespace）
- 多個 namespace 可塞同一檔（依函式庫支援度而定）

#### ⚠️ Namespace ≠ Key 前綴

容易混淆但本質不同：

```js
// 純前綴（命名約定，無實際隔離）
t('candidate.name')    // key 是 "candidate.name"，仍在全域池
t('requisition.name')  // 撞名風險取決於是否有人寫過同名 key

// 真正的 namespace
t('candidate:name')    // namespace=candidate, key=name
t('requisition:name')  // 完全隔離，函式庫機制層級保證不撞
```

純前綴方案常見於**不支援 namespace 的函式庫**（如 react-intl 靠 message ID 命名約定），仰賴開發者紀律避免撞名；namespace 則是函式庫層級的隔離。

#### Namespace 不只防撞名

兩個常被忽略的副效用：

- **Lazy load 的天然邊界**：一個 namespace 一次載入，首屏只下載必要的翻譯。例如 `useTranslation('candidate')` 進入候選人頁時才載入該 namespace，職缺管理員永遠不會下載到候選人介面的翻譯。
- **翻譯工作的切割單位**：一個 namespace 可獨立發包給一位譯者或交給翻譯平台處理，不會干擾其他 namespace 的版本。

#### 何時可以省略 namespace

小專案（< 200 條 key、單一團隊）可以全塞在一個 namespace（慣稱 `common` 或 `translation`），換來：
- 開發者不需要思考 key 該分到哪個 namespace
- 翻譯檔結構單純

代價是當專案長大、key 數量爆增時，撞名問題會浮現。**shared-lib wrapper 就是這種案例**：所有翻譯被合併到單一 namespace，加上強制 lowercase，導致 `term/zh.js` 與 `message/zh.js` 的同名 key 互相覆蓋（細節見「自寫 wrapper」段落）。

函式庫對 namespace 的支援差異（後面深入比較）：
- **i18next**：一級公民，`t('ns:key')` 語法、`useTranslation(ns)` 選擇載入
- **react-intl (FormatJS)**：沒有原生 namespace，靠 message ID 命名約定（如 `candidate.name`）
- **Lingui**：以 message ID 為核心，可用編譯期巨集自動產生 ID
- **vue-i18n**：支援 namespace 但語法略不同（透過 `useScope` 與 message structure）

### Key 命名策略

決定怎麼命名翻譯 key 是 i18n 架構最早的分叉點之一，會持續影響後續所有翻譯維護成本。主流有三種純策略加一種混合策略。

#### 三種純策略

**A. Namespace 隔離**

```js
t('candidate:name')    // 候選人姓名
t('requisition:name')  // 職缺職稱
```

靠函式庫的 namespace 機制隔離，相同 key 在不同 namespace 完全不會撞。

**B. 功能前綴 key**

```js
t('candidate.name')    // 候選人姓名
t('requisition.name')  // 職缺職稱
```

純命名約定，key 仍平攤在全域池。前綴只是視覺上的分組，無實際隔離機制。

**C. 自然語言 key**

```js
t('Candidate Name')        // 候選人姓名
t('Requisition Job Title') // 職缺職稱
```

用完整英文短語當 key，英文版翻譯通常等於 key 本身（fallback 自動完成）。

#### 三策略優缺點

| 策略 | 優點 | 缺點 |
|------|------|------|
| **Namespace 隔離** | 機制保證不撞、lazy load 友善、翻譯切割清晰 | 函式庫需支援、開發者要記得指定 ns |
| **功能前綴** | 相容所有函式庫、零學習成本 | 純紀律無保障、key 易長、靜態工具難分析 |
| **自然語言** | 可讀性最高、英文 fallback 自動、適合翻譯平台 | 英文版 key === value 重複、改文案要連動改程式碼、非英文母語團隊使用不便、**歧義詞（apply / system / submit 等）仍會撞名**（詳見後段反例） |

#### D. 混合策略（實務最常見）

**通用詞用自然語言 key + 業務詞用 namespace**：

```js
t('Save')                  // 通用按鈕，全站共用
t('Cancel')                // 通用按鈕
t('candidate:name')        // 業務特化
t('requisition:status')    // 業務特化
```

優點是兩種策略各自發揮所長 — 通用詞享受自然語言 key 的高可讀性、業務詞享受 namespace 的隔離保障。代價是團隊需要對「什麼算通用詞」有共識。

**通用詞判定原則**（三條都符合才算）：
1. **語義跨業務一致** — 在所有功能下中文翻譯都相同（`Save` 永遠是「儲存」）
2. **沒有業務歧義** — 不是 `apply` / `submit` / `system` 這種多義詞
3. **是純粹的 UI 動作或狀態詞** — 不涉及業務名詞

**起手清單範本**（可直接複製到專案 `i18n-common-words.md`）：

| 類別 | 通用詞 |
|------|--------|
| 動作 | save, cancel, close, confirm, delete, edit, add, remove, search, reset, copy, download, upload |
| 導航 | back, next, previous, home |
| 狀態（純 UI） | loading, success, error, warning, info |
| 是非 | yes, no |

**邊界陷阱詞（看似通用、實則歧義 — 一律走 namespace）**：

| 詞 | 歧義 | 必須語境化 |
|---|---|---|
| `apply` | 應徵 vs 套用 | `apply-requisition`、`apply-settings` |
| `submit` | 送出 form vs 送出簽核 | `submit-form`、`submit-approval` |
| `system` | 系統管理 vs 甄試系統 | `system-management`、`system-notification` |
| `accept` / `reject` | offer / interview / approval 都用 | 加業務前綴 |

不在白名單的詞**一律走 namespace**。新增白名單需 PR review。

#### 決策矩陣

| 你的條件 | 建議策略 |
|---------|---------|
| 函式庫不支援 namespace（如 react-intl） | 前綴 或 自然語言 |
| 多語系 ≥ 3 種、需 lazy load 控制 bundle 大小 | Namespace 隔離 |
| 翻譯團隊英文母語、用 Crowdin / Lokalise | 自然語言 key |
| 專案小（< 300 keys）/ 早期 prototype | 混合策略起步，未來再演進 |
| 專案大、功能模組結構穩定 | Namespace 隔離 |
| 不確定怎麼選 | **預設混合策略** — 容易演進到任一純策略 |

#### Key 規格細節

實作層的選擇也會影響長期維護：

- **英文 vs 中文 key**：英文工具友善（lint、自動 extract、IDE autocomplete 都好處理）；中文可讀但編譯期工具普遍不支援，**強烈建議英文 key**
- **大小寫與分隔符**：`camelCase` / `PascalCase` / `kebab-case` 都可，**選一種團隊一致**就好。自然語言 key 自然帶空格（`'Candidate Name'`），前綴/namespace 通常 `camelCase` 或 `kebab-case`
- **⚠️ case-sensitive 陷阱**：若函式庫或自訂 wrapper 強制 lowercase，`Apply` 與 `apply` 視為同 key — 命名策略要假設 case-insensitive 才安全。設計新專案時優先選擇 case-sensitive 函式庫（i18next、react-intl、Lingui 預設皆是）

#### 純自然語言 key 也救不了沒有 namespace

需要特別點出的反例：**shared-lib wrapper 採「自然語言 key + 無 namespace + 強制 lowercase」三重組合**，結果是自然語言 key 完全無法防撞 — `Apply` 在候選人功能是「應徵」、在主試官對話框是「套用」，因為沒有 namespace 隔離，後載入者覆蓋前載入者（細節見「自寫 wrapper」段落）。

**啟示**：自然語言 key 看起來「不太可能撞」是錯覺，業務字彙的歧義（apply / submit / system / name 等）會在大型專案中浮現。沒有 namespace 機制時，必須**主動避開歧義詞**、使用更長的語境化短語（如 `'Apply Requisition'` 而非 `'Apply'`）。

### 檔案組織策略

翻譯檔的組織決定兩件事：開發體驗（找 key 容易嗎？新增功能要動哪幾個檔？）與 runtime 行為（首屏要下載多少 KB？）。決策有兩個獨立維度：

- **維度 1：拆檔粒度** — 單檔 / 按 namespace 拆 / 按頁面 co-location
- **維度 2：目錄外層** — 按語系優先 vs 按 namespace 優先

#### 三種主流結構

**A. 單檔扁平**

```
locales/
├── zh.json
└── en.json
```

```json
// locales/zh.json
{
  "Save": "儲存",
  "Cancel": "取消",
  "Candidate Name": "候選人姓名",
  ...500 keys...
}
```

適合小專案（< 300 keys）、結構單純、import 容易；擴展性差、bundle 全部下載、撞名難解。

**B. 按 namespace 拆（中大型主流）**

```
locales/
├── zh/
│   ├── common.json
│   ├── candidate.json
│   └── requisition.json
└── en/
    ├── common.json
    ├── candidate.json
    └── requisition.json
```

```js
// 元件內使用
const { t } = useTranslation('candidate');
t('name');  // → 從 locales/zh/candidate.json 載入
```

優點：lazy load 友善、譯者切割清晰、找 key 容易。缺點：要在功能規劃階段決定 namespace 邊界（哪些算 candidate、哪些算 common）。

**C. Co-location（元件級隔離）**

```
components/
└── Candidate/
    ├── index.jsx
    └── locales/
        ├── zh.json
        └── en.json
```

```js
// 元件目錄內 import
import zh from './locales/zh.json';
i18n.addResourceBundle('zh', 'candidate', zh);
```

優點：元件搬移時翻譯一起跟著走、適合元件套件分發（如 npm package）。缺點：跨元件共用詞難處理、全站搜尋 key 變麻煩。

#### 三種結構優缺對照

| 結構 | 優點 | 缺點 | 適用 |
|------|------|------|------|
| 單檔扁平 | 結構單純、import 容易 | 擴展性差、bundle 全載、撞名難解 | 小專案（< 300 keys） |
| 按 namespace 拆 | lazy load 友善、譯者切割清晰、找 key 容易 | 要決定 namespace 邊界 | 中大型專案 |
| Co-location | 元件搬移翻譯跟著、適合元件分發 | 跨元件共用難、全站搜尋麻煩 | 元件套件 / 微前端 |

#### 目錄結構慣例

按 namespace 拆檔時，目錄外層有兩種寫法：

| 結構 | 範例 | 適用 |
|------|------|------|
| **語系優先**（主流） | `locales/{lang}/{namespace}.json` | i18next-http-backend 預設、適合按語系發包翻譯 |
| **namespace 優先**（少見） | `locales/{namespace}/{lang}.json` | 適合按模組發包，但工具支援度低 |
| **Co-location 的混合** | `Component/locales/{lang}.json` | 單 namespace 視角，namespace 由元件名隱含 |

預設選**語系優先**，除非有強烈的「按模組發包」需求。

#### Lazy load 時機

決定哪些 namespace 首屏載入、哪些按需載入：

| 策略 | 適用 | 注意 |
|------|------|------|
| **全部首屏載入** | 小專案（< 300 keys） | 簡單、無 loading 閃爍 |
| **預載核心 + 按需載業務** | 中大型專案 | `common` namespace 首屏、業務 namespace 進頁面才載 — 平衡方案 |
| **完全按需載入** | 超大型多模組 | 首屏最快但有 loading 閃爍風險，需設計 Suspense fallback |

i18next 用 `i18next-http-backend` 自動處理 lazy load；react-intl / Lingui 通常手動 `import()` 動態載入後 dispatch 到 store。

#### 演進路徑

實際專案常從簡單結構長大到複雜結構，演進有跡可循：

**演進 1：單檔 → 按 namespace 拆**

觸發點：撞名問題出現 / bundle 過大 / 多人協作衝突。

**Namespace 邊界判斷原則**（決定每個 key 該放哪個 namespace）：
- **以「同時使用的頁面範圍」分組** — 候選人管理頁的詞放 `candidate`、職缺管理頁的詞放 `requisition`
- **跨多個頁面共用的詞拉到 `common`** — 按鈕、狀態、通用提示
- **錯誤訊息獨立成 `error`** — 通常跨頁面、來源是後端，獨立 namespace 便於統一處理

步驟：
1. 依上述原則分類所有 key，產出 mapping 清單（key → namespace）
2. 依 mapping 把扁平檔拆成多檔（不動載入機制，仍全載），跑一輪測試確認沒漏
3. 改呼叫端：`t('Save')` → `useTranslation('common'); t('Save')`，模組逐個改
4. 預載 `common` + `error`，業務 namespace 設 lazy load

**演進 2：按 namespace 拆 → 加上 co-location**

觸發點：要把某個元件抽成獨立套件分發。步驟：
1. 把該元件的翻譯 key 移到 `Component/locales/`
2. 元件初始化時 `addResourceBundle` 註冊到對應 namespace
3. 全域翻譯檔保留 fallback（避免外部使用者沒帶翻譯時崩潰）

#### 檔案格式取捨

| 格式 | 優點 | 缺點 | 何時用 |
|------|------|------|--------|
| **JSON** | 最通用、所有主流函式庫 + Crowdin/Lokalise 都支援 | 不能寫註解 | **預設選擇** |
| **JS / TS** | 可寫註解、可 import 互相組合、可動態生成 | 翻譯平台不友善（不是純資料），譯者無法直接編輯 | 開發團隊自己維護翻譯時 |
| **YAML** | 可寫註解、結構更人性 | React 生態支援度低於 JSON | 後端為主、跨 stack 共用翻譯時 |
| **PO (gettext)** | pluralization / context 機制最完整、翻譯工作流歷史標準 | React 生態少見、工具支援需額外設定 | 老牌跨平台產品、與 Django/PHP 整合 |

對 React 專案的建議：**預設 JSON**。若需要寫註解，可用 JSONC（VS Code 支援）或在 key 命名中帶上情境（如 `'candidate.name.tooltip'`）。

#### 推薦預設方案

新中型 React 專案沒有特殊條件時的安全起點：

> **JSON + 按 namespace 拆 + `locales/{lang}/{namespace}.json` + 預載 `common`、其他 namespace lazy load**

這個組合：
- 涵蓋 80% 中型專案的長期需求
- 任一條件變化時都能漸進演進（拆更細、加 co-location、改 lazy 策略）
- 工具鏈支援最完整（i18next-parser、Crowdin、TypeScript 型別生成都 work）

從這個預設出發，遇到具體限制再針對性調整 — 避免一開始就為了「靈活性」過度設計。

## 主流函式庫比較

### i18next / react-i18next

**定位**：React 生態的事實標準。功能最完整、社群最大、plugin 最多。

**核心特色**
- Namespace 是一級公民（`t('ns:key')` 語法）
- Lazy load 內建（搭配 `i18next-http-backend`）
- Plugin 生態豐富：parser（自動 extract）、icu（ICU MessageFormat）、locize（翻譯平台整合）
- 支援 interpolation、pluralization、context、nested key

| 優點 | 缺點 |
|------|------|
| 功能完整、文件齊全 | API surface 大、初學者設定多 |
| 社群最大、Stack Overflow 答案最多 | Bundle 比 Lingui / react-intl 大（~40KB gzipped） |
| 各種後端、檔案格式、平台整合都有 plugin | 預設啟用的特性（key separator、ns separator）若不關掉容易誤觸 |

```js
import i18next from 'i18next';
import { useTranslation } from 'react-i18next';

i18next.init({
  lng: 'zh',
  ns: ['common', 'candidate'],
  resources: {
    zh: {
      common: { Save: '儲存' },
      candidate: { name: '姓名' },
    },
  },
});

// 元件內
const { t } = useTranslation('candidate');
t('name');          // → 姓名（namespace 內）
t('common:Save');   // → 儲存（指定 namespace）
```

**何時選**：需要 namespace 隔離、lazy load、多後端載入、或不確定該選什麼時的安全預設。

### react-intl (FormatJS)

**定位**：Meta / Yahoo 出品，主打 ICU MessageFormat 標準。元件式 API。

**核心特色**
- ICU MessageFormat 是業界 pluralization / 性別 / 複雜變數的標準語法
- 無原生 namespace 機制 — 靠 message ID 命名約定（如 `candidate.name`）
- 強型別 TypeScript 支援、CLI 工具 `formatjs extract` 自動掃描
- 元件式 API：`<FormattedMessage id="..." />`，也提供 hook 版

| 優點 | 缺點 |
|------|------|
| ICU pluralization 業界標準（多語系複數規則最完整） | 無 namespace，需要靠 ID 命名紀律 |
| TypeScript 型別與 extract 工具最成熟 | ICU 語法學習曲線（`{count, plural, one {# item} other {# items}}`） |
| Meta / Yahoo 大型專案實戰驗證 | 元件式 API 在 string-only 情境（如 title、alt）較尷尬 |

```jsx
import { FormattedMessage, useIntl } from 'react-intl';

// 元件式
<FormattedMessage id="candidate.name" defaultMessage="Name" />

// Hook 式
const intl = useIntl();
intl.formatMessage({ id: 'candidate.name', defaultMessage: 'Name' });
```

**何時選**：重視 ICU pluralization、TypeScript 型別、有自動 extract 工作流；可接受用命名約定取代 namespace。

### Lingui

**定位**：「寫程式碼即翻譯」開發體驗最好的選項。編譯期巨集自動產 message ID。

**核心特色**
- Babel/SWC 巨集在編譯期自動產生 message ID（不用手動命名 key）
- 自帶 `lingui extract` 工具掃描程式碼
- 支援 ICU MessageFormat
- Bundle 小（runtime 約 ~10KB gzipped）

| 優點 | 缺點 |
|------|------|
| 維護成本最低（key 由原始文字自動生成） | 依賴 Babel/SWC plugin，工具鏈設定較複雜 |
| TypeScript 友善、IDE 體驗好 | 編譯期魔法增加除錯難度（看程式碼推不出最終 key） |
| Bundle 小、效能佳 | 社群比 i18next 小、文件覆蓋面較窄 |

```jsx
import { Trans, t } from '@lingui/macro';

// 巨集寫法（編譯後產出 message ID）
<Trans>Candidate Name</Trans>
t`Save`;

// 編譯後 ↓
// i18n._('Candidate Name')
// i18n._('Save')
```

**何時選**：團隊熟悉現代工具鏈、要最低維護成本、可接受編譯期魔法。

### vue-i18n

**定位**：Vue 生態的事實標準。設計與 i18next 同代際，整合 Vue Composition API。

**核心特色**
- 官方維護（vuejs/vue-i18n）、文件中英齊全
- Namespace 透過 message structure 實現（巢狀物件 + scope）
- Composition API 友善（`useI18n()`）
- 支援 interpolation、pluralization、datetime/number formatting

| 優點 | 缺點 |
|------|------|
| Vue 整合最深、官方掛保證 | 只能用於 Vue 專案 |
| API 簡單、學習曲線短 | 大型專案的 lazy load 需要手動組 |
| TypeScript 型別佳（v9+） | 跨 Vue 2/3 版本遷移有 breaking change |

```js
import { useI18n } from 'vue-i18n';

const { t } = useI18n();
t('candidate.name');  // → 姓名
```

**何時選**：Vue 專案。沒有別的選擇也不需要 — 是該生態的合理預設。

### 自寫 wrapper（警示對照組）

**定位**：通常是「省事不引入依賴」的決定，長期後悔的案例多。

**為什麼有人這麼做**
- 避免大型依賴（誤判：i18next gzipped 約 40KB，bundle 影響其實有限）
- 業務有特殊需求（多數情境其實可用現有函式庫的 plugin 解決）
- 早期專案決定（當時 i18n 生態未成熟，但這個藉口已不適用 2020 年後的專案）

**典型風險**
- 漏掉 namespace 機制 → 大型化後撞 key
- 漏掉 interpolation / pluralization → 之後要補就是 breaking change
- 自製的 key 處理規則（如強制 lowercase）破壞主流慣例 → 工具鏈全部不能用
- 沒人寫文件 → 後人不知道 wrapper 的隱藏行為（如本專案 shared-lib 的 lowercase 機制）

**真實案例：shared-lib wrapper**

本專案前端的自寫 wrapper（`src/lib/shared-lib/core/i18n.js`）在 i18next 上加了三個「自製規則」：

```js
const key = text.toLowerCase();         // 強制 key lowercase
const namespace = "translation";        // term/message/article 合併為單一 namespace
const keySeparator = false;             // 關閉 i18next 預設分隔符
const nsSeparator = false;
```

後果：
- `term/zh.js: Apply="應徵"` 被 `message/zh.js: Apply="套用"` 覆蓋（commit `7b262553f`，2026-04-28）
- Workbench 的 `"System"` key 被甄試 dialog 既有的 `"System"` 覆蓋（commit `042bdea01`，2026-04-16）

**遷移成本**

自寫 wrapper 往主流函式庫遷移很痛：
- Key 命名假設不同（lowercase / case-sensitive、扁平 / 巢狀）
- 檔案格式不同（JS module / JSON / PO）
- 整個 codebase 的 `t()` / `translate()` 呼叫都要審視

通常只能採「new code 用新方案、legacy 維持 wrapper」的雙軌共存，技術債就此成形。

**何時可以接受**

幾乎沒有。只有「絕對不會擴張、絕對不會有第三方參與翻譯」的玩具專案 — 而玩具專案連 i18n 都不需要。

> 給未來自己的提醒：**任何「自寫 i18n 比較簡單」的論證都應該被質疑**。i18next 的設定門檻 < 30 分鐘，換來的是整個生態的工具鏈支援。

## 選擇指南

把前面的決策元素整合成可執行的提問流程。分兩種情境：新專案從零選 vs 接手既有專案。

### 新專案決策樹

```
框架是？
├── Vue ───────────────────────── vue-i18n
├── 其他（Angular / Svelte 等） ── 各生態標準
└── React ──────────────────────► 繼續

  有不可妥協需求嗎？
  ├── ICU pluralization 是硬需求 ─── react-intl 或 Lingui
  ├── 編譯期自動 extract 是硬需求 ── Lingui
  ├── 最完整文件 / 最大社群 ─────── i18next
  └── 無特別硬需求 ──────────────► 繼續

    TypeScript 投入程度？
    ├── 全面 TS、重視型別 ──────── react-intl / Lingui 略勝
    └── 不關心型別 ─────────────► 繼續

      翻譯平台整合需求？
      ├── 用 Crowdin / Lokalise ── 三者皆可，i18next 整合最直接（官方 locize）
      └── 無 ────────────────► 繼續

        🎯 預設選擇：i18next
```

**為什麼 i18next 是兜底預設**：文件最完整、社群最大、plugin 生態最廣、新人上手最快、出問題最容易 Google 到答案。除非有具體需求把你推離預設，否則它是安全選擇。

### 接手既有專案的盤點流程

接手陌生 codebase 時，依序執行：

**Step 1：找出函式庫**
```bash
grep -E 'i18next|react-intl|lingui|vue-i18n' package.json
```

**Step 2：找 wrapper**
```bash
find src -name 'i18n*' -type f
```
讀任何 wrapper 的設定，特別注意是否改動了：
- `keySeparator` / `nsSeparator`（影響巢狀 key 與 namespace 寫法）
- 強制 `toLowerCase()` 或其他 key normalization
- 預設 namespace 設定

**Step 3：看翻譯檔結構**
```bash
ls src/nls src/locales src/i18n 2>/dev/null
```
確認是單檔扁平、按 namespace 拆、還是 co-location。

**Step 4：驗證 case-sensitivity**

寫個臨時測試確認大小寫是否視為同 key：
```js
console.log(t('Foo') === t('foo'));  // false = case-sensitive，true = 被強制 lowercase
```

**Step 5：掃過 i18n 相關 bug 史**
```bash
git log --all --oneline | grep -iE 'i18n|translation|翻譯'
```
過去踩過的雷常會再踩 — 從 commit message 與 fix diff 學到 wrapper 的隱藏行為。

### 該不該替換既有方案？

替換 i18n 函式庫是大工程，要審慎評估。

**替換成本估算**

```
替換成本 ≈ key 數量 × 平均改動成本（1-3 處呼叫 / key）
        + 翻譯檔格式轉換
        + 全站測試
```

**觸發替換的條件**（任一達成就值得認真考慮）
- 撞 key 災難已成日常（如 shared-lib 案例）
- 翻譯量爆增、bundle 過大、無法 lazy load
- 工具鏈無法接入（要 TypeScript 型別、要自動 extract、要翻譯平台）

**不該替換的條件**（任一達成就建議維持）
- 翻譯量小（< 500 keys）— 痛還沒到、收益小於成本
- 即將下線的舊系統 — ROI 為負
- 沒人有時間做全面測試 — 半套替換比不替換更糟

> **註：兩種「< 500 / < 300 keys」門檻的差別**：
> - **< 300 keys = 小專案規模分界**（決定結構複雜度：單檔扁平就夠了）
> - **< 500 keys = 替換 ROI 門檻**（決定是否值得投入工程資源換函式庫）
>
> 兩者語境不同 — 300-500 keys 區間屬於「結構該升級但替換還不值得」的灰色地帶。

**折衷方案：雙軌共存**

Legacy 維持、新程式碼用新方案。但**要設清楚邊界**，否則會變第三條軌（新舊混用、誰也說不清）：

- 邊界方案 A：以模組劃分（如「新建立的 module 用 new i18n」）
- 邊界方案 B：以時間劃分（如「2026 後新增 key 一律進新系統」）
- 邊界方案 C：以元件層劃分（如「page 級用新、shared component 維持舊」）

任選一個並寫進 CONTRIBUTING.md，禁止臨時起意決定。

### 常見搭配（快速套用清單）

| 情境 | 推薦組合 |
|------|---------|
| 中型 SPA、TypeScript、無翻譯平台 | **i18next + i18next-parser + JSON** |
| 大型 SPA、ICU pluralization、有翻譯平台 | **react-intl + FormatJS CLI + Crowdin** |
| 現代工具鏈、低維護、TypeScript | **Lingui + lingui extract + JSON** |
| Vue 專案 | **vue-i18n + JSON** |
| 老 codebase、預算少、不替換 | **既有 wrapper + 補撞 key 防護**（lint / pre-commit / Claude skill 如 專用的翻譯檢查流程） |

## 翻譯檔結構建議

把前面的概念轉成可直接複製的模板與避雷清單。

### 四種規模的目錄模板

**玩具專案（< 50 keys）**

```
src/
└── i18n.json
```

或最低限度的雙語支援：

```
src/locales/
├── zh.json
└── en.json
```

不分 namespace、不拆檔、不 lazy load。任何超過此規模的設計都是過度工程。

**小專案（< 300 keys）**

```
src/locales/
├── zh.json
└── en.json
```

```json
// locales/zh.json
{
  "common": {
    "Save": "儲存",
    "Cancel": "取消"
  },
  "candidate": {
    "name": "姓名"
  }
}
```

仍是單檔，但用物件巢狀預先分組（用 `common` / `candidate` 等 group）。當功能爆發時可無痛拆檔 — 巢狀結構直接對應到分檔結構。

**中型（300-2000 keys，中型專案等級）**

```
src/locales/
├── zh/
│   ├── common.json        # 全站共用按鈕、狀態
│   ├── error.json         # 錯誤訊息
│   ├── candidate.json
│   ├── requisition.json
│   └── interview.json
└── en/
    ├── common.json
    ├── error.json
    ├── candidate.json
    ├── requisition.json
    └── interview.json
```

- 按 namespace 拆，目錄外層用語系優先
- `common` + `error` 首屏載入
- 業務 namespace（candidate / requisition / interview）lazy load
- 配合 i18next-parser 自動 extract

**大型（> 2000 keys、多模組）**

```
src/
├── locales/                        # 全站共用
│   ├── zh/
│   │   ├── common.json
│   │   └── error.json
│   └── en/
│       ├── common.json
│       └── error.json
└── modules/
    ├── Candidate/
    │   ├── index.jsx
    │   └── locales/
    │       ├── zh.json
    │       └── en.json
    └── Requisition/
        ├── index.jsx
        └── locales/
            ├── zh.json
            └── en.json
```

兩層架構：共用翻譯集中、模組翻譯 co-location。適合：
- 模組級獨立部署 / 抽成 npm 套件
- 多團隊各自負責不同模組
- 微前端架構

### 反模式清單

| 反模式 | 後果 |
|--------|------|
| 單檔超過 2000 行 | 人類難 grep、git diff 噪音大、合併衝突頻繁 |
| 巢狀深度 > 3（`a.b.c.d.e`） | 工具難掃、IDE 補全失效、key 路徑記不住 |
| 檔名混語系與 namespace（`zh-candidate.json`） | 工具無法解析 lang/namespace 兩個維度 |
| 同 key 在不同 namespace 的中文不一致但英文一致 | 修文案要改 N 個地方，遲早漏改 |
| JSON 內偽註解（`"_comment": "..."`） | 會被視為 key 嘗試翻譯，亂顯示。要註解就改 JSONC 或 JS |
| 動態組合 key（``t(`candidate.${field}`)``） | extract 工具掃不到，被視為「未使用」誤刪。**改寫**：用顯式 map（`const labelMap = { name: t('candidate:name'), email: t('candidate:email') }; labelMap[field]`），讓所有 key 都是靜態字串 |
| 翻譯檔放在 `public/` | 用戶可直接下載所有翻譯（含未上線文案、內部術語） |
| 沒有英文 fallback | 缺翻譯時 UI 顯示 raw key 或崩壞 |
| 不版本化翻譯檔 | rollback 程式碼時翻譯對不上 |

### 長期維護工作流

#### 新增 namespace 的 SOP

判斷該新建獨立 namespace 還是加進既有：

| 條件 | 建議 |
|------|------|
| 預估 key 數 < 50 | 加進既有最相關的 namespace |
| 預估 key 數 ≥ 50 / 功能模組獨立性高 | 新建獨立 namespace |
| 跨多個既有 namespace 共用 | 拉到 `common` 或新建 shared namespace |

#### Key 棄用流程

避免「程式碼已不用、翻譯檔遺留」的腐爛 key：

1. 程式碼移除呼叫
2. `grep -r "key-name" src/` 確認無遺留使用
3. 在翻譯檔標記 `// DEPRECATED 2026-MM-DD` 註解（或加 `__deprecated__` 前綴）
4. 下一個 release 從翻譯檔移除
5. 通知譯者該 key 已下線（避免他們繼續翻新值）

#### 翻譯版本管理

- 翻譯檔與程式碼**放同一個 repo**（避免版本漂移）
- Release tag 時翻譯檔一起被 tag
- Rollback 程式碼時翻譯也回滾（不該分開）
- 若用翻譯平台（Crowdin），設定「同步至特定 branch」而非「直接覆寫 main」

## 周邊議題

### 日期/數字格式化

日期、數字、貨幣是「locale-aware formatting」不是「翻譯字串」 — 不該塞進翻譯檔，該用格式化 API。

- **原生 `Intl.DateTimeFormat` / `Intl.NumberFormat`** 已涵蓋多數場景，無需額外套件
- 複雜日期算術用 **date-fns / dayjs** 並搭配其 locale 套件
- 各 i18n 函式庫整合：
  - i18next：靠 formatter plugin（`i18next.use(Formatter)`）
  - react-intl：`<FormattedDate>`、`<FormattedNumber>` 元件
  - Lingui：`i18n.date(value)` / `i18n.number(value)`

```js
// 原生 Intl，多數情況夠用
new Intl.DateTimeFormat('zh-TW', { dateStyle: 'long' }).format(new Date());
// → 「2026年5月21日」

new Intl.NumberFormat('zh-TW', { style: 'currency', currency: 'TWD' }).format(1234567);
// → 「NT$1,234,567」
```

**何時投資**：多語系專案上線前；單語系專案可延後處理。

### RTL 排版

支援阿拉伯文、希伯來文、波斯文等右至左語系。比想像中影響範圍大 — 整個版面要鏡像。

- **HTML 層級**：`<html dir="rtl">` 或元素級 `dir="rtl"`，切換語系時動態切換 `document.documentElement.dir`
- **CSS 層級**：用 logical properties（`margin-inline-start` 而非 `margin-left`、`padding-inline-end` 而非 `padding-right`），現代瀏覽器全支援
- **整體鏡像**：不只文字方向 — icon 位置、動畫方向、列表縮排、圖示順序都要跟著鏡像
- **UI 庫支援**：MUI / Chakra UI 有 RTL theme provider，可一鍵切換

**何時投資**：產品確定要進中東 / 北非市場；其他情境可不處理。

### TypeScript 型別補完

讓「用錯翻譯 key」變成編譯期錯誤，而非 runtime 「key not found」。

各函式庫支援度：

| 函式庫 | 型別生成方式 | 體驗 |
|--------|-------------|------|
| i18next | `i18next.d.ts` + module augmentation | 需手動設定，可由 i18next-parser 輔助 |
| react-intl | messages object 直接帶型別 | 最佳，messages 即型別來源 |
| Lingui | 編譯期巨集帶型別資訊 | 自動，無需設定 |

**設定步驟**（i18next 範例）：
1. build-time 工具掃翻譯檔產生 `keys.d.ts`
2. 透過 module augmentation 告訴 i18next 可用 key
3. 把 type 生成步驟加入 CI（每次翻譯檔變動都重產）

**注意**：型別生成必須列入 CI，否則新增 key 卻沒重產型別會錯失保護 — 變成「型別永遠落後實際翻譯一個 commit」。

**何時投資**：團隊全面 TypeScript、且 key 數量超過小專案門檻（指標：~300 keys，視團隊紀律而定 — key 紀律好可延後，紀律差可提前）。

### 自動 key extraction

從程式碼掃描所有 `t(...)` / `<FormattedMessage>` 呼叫，自動產出未翻譯 key 清單。

主流工具（綁定其函式庫）：
- **i18next-parser** — i18next 用
- **@formatjs/cli** — react-intl 用
- **lingui extract** — Lingui 用

**CI 整合**：把 extract 結果與既有翻譯檔 diff，差異即「未處理 key」，直接擋住 build：

```bash
# 範例：i18next-parser
npx i18next-parser --config i18next-parser.config.js
git diff --exit-code locales/  # 有差異就失敗
```

**限制**：靜態分析無法處理動態組合 key（呼應前面反模式：``t(`candidate.${field}`)`` 掃不到）。要使用 extract 工具就要禁止動態 key。

**何時投資**：翻譯量 > 500 keys 之後，手動維護易遺漏；早期專案可省略。

> 與 TypeScript 型別補完的 ~300 keys 門檻不同：型別補完防的是「**用錯既有 key**」（嚴重度高、頻率低），key 數越多誤用機率越高；extraction 防的是「**新增 key 卻忘記補翻譯**」（嚴重度中、頻率隨開發節奏），與 key 新增速率相關。兩者獨立成本/收益曲線，所以閾值不同。

### 翻譯平台整合

讓譯者用 GUI 翻譯，而非直接編輯 JSON / Git。提供 review、approval、譯者協作、翻譯記憶（同樣的英文不重複翻）。

主流平台：

| 平台 | 特色 |
|------|------|
| **Crowdin** | 開源專案免費、社群翻譯友善 |
| **Lokalise** | 商業級、API 完整、CI 整合佳 |
| **locize** | i18next 官方、無縫整合 |
| **Phrase** | 老牌商業方案、多 stack 支援 |

**典型工作流**：
1. 開發者 push 來源語系（通常英文）翻譯檔到平台
2. 平台分派譯者翻譯
3. 譯者透過 GUI 提交譯文
4. CI / webhook 拉回各語系檔案到 repo

**關鍵決策：single source of truth 在哪？**

定義 SoT 的關鍵是「**新增 key 由誰主導**」與「**衝突時誰勝**」：

| 方向 | 新增 key 流向 | 衝突勝者 | 適用 |
|------|------------|---------|------|
| **Git 為主** | 開發者在 git push 新 key → 平台 webhook 同步給譯者翻譯 → 譯者提交譯文回 git（透過 PR 或自動 commit） | git 勝 | 開發團隊主導翻譯流程 |
| **平台為主** | 譯者在平台新增 key → 平台 webhook 同步 git → 開發者 pull 後用 | 平台勝 | 翻譯團隊主導、譯者 > 開發者 |
| **雙向同步** | 兩邊都可新增 | 視同步邏輯而定（通常後到勝） | **不建議** — 衝突解決惡夢 |

「Git 為主」的工作流範例：
1. 開發者在程式碼 push 新 key（含預設英文 value）
2. CI 把新 key 推到平台，平台分派譯者翻譯
3. 譯者提交譯文 → 平台自動開 PR 到 git（或 CI 拉回 commit）
4. 開發者 review PR、merge

**預設選「Git 為主」**，並在 CONTRIBUTING.md 明文寫死方向，禁止繞過流程直接編 git 翻譯檔。

**何時投資**：有專職譯者 / 多語系 ≥ 3 / 翻譯量 > 1000 keys。

## 備註

### 真實踩雷彙整

本專案中與 i18n 相關的踩雷紀錄：

| Commit | 日期 | 問題 | 教訓 |
|--------|------|------|------|
| `7b262553f` | 2026-04-28 | `term/zh.js: Apply="應徵"` 被 `message/zh.js: Apply="套用"` 覆蓋，候選人「應徵」按鈕變「套用」 | 自然語言 key 在歧義詞（apply / submit / system / name）上不夠安全；沒有 namespace 機制時必須主動避開歧義詞、改用更長的語境化短語 |
| `042bdea01` | 2026-04-16 | Workbench 的 `"System"` key 被甄試 dialog 既有的 `"System"` 覆蓋為「系統」 | 通用單字（system / status / type）撞名風險最高，應一律加 feature 限定詞（`"System Management"`、`"Approval Status"`） |

共同模式：shared-lib wrapper 把 term/message/article 合併為單一 namespace + 強制 lowercase 的設計取捨，讓自然語言 key 失去本應有的「歧義隔離」保護。當前的防護與重構規劃詳見「[本專案的具體重構建議](#tas-專案的具體重構建議)」段。

### 進階閱讀

**官方文件**
- i18next: https://www.i18next.com
- react-i18next: https://react.i18next.com
- react-intl / FormatJS: https://formatjs.io
- Lingui: https://lingui.dev
- vue-i18n: https://vue-i18n.intlify.dev
- MDN — `Intl` API: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl

**翻譯平台**
- Crowdin: https://crowdin.com
- Lokalise: https://lokalise.com
- locize (i18next 官方): https://locize.com

### 相關 Claude skill

- `i18n-translate` — 本專案前端翻譯檢查與撞 key 防護，於 `~/.claude/skills/i18n-translate/` 定義；可被 `finalize` 流程自動呼叫

