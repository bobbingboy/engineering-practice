# Engineering Notes & Agent Skills

兩部分：一套實際在用的 Claude Code 開發流程 skill，以及支撐這些判斷的技術筆記。

內容抽自日常工作環境，已移除客戶名稱、主機位址與內部產品名，保留完整的技術脈絡與判斷邏輯。

---

## skills/

### 收尾流程編排

功能實作完成到可以送出之間，有一段順序有講究的工作。`finalize` 把它連同護欄一起編碼成流程：

```
finalize
  ├─ 0.5 端到端驗證   （改動落在單元測試盲區時）
  ├─ 1   歸檔          （OpenSpec change，若有）
  ├─ 2a  code-quality      ─┐
  ├─ 2b  code-correctness   ├─ 各自跑在獨立 sub-agent
  ├─ 2c  第二意見（可選）    ─┘
  ├─ 2.5 i18n 檢查     （改動含翻譯檔時）
  ├─ 3   react-perf-check  （改動含 React 前端時）
  ├─ 4   commit 範圍確認 → 等授權
  ├─ 5   commit
  └─ 6   push          （明確授權後）
```

**為什麼三個審查跑在獨立 sub-agent**：同一個 context 裡剛寫完程式碼的 agent 去 review 自己的產出，會系統性地看不見自己的假設。換獨立 context、換規則集，才有機會抓到。

**為什麼順序不能換**：review 必須在 commit 前，修復才能一起進 commit；歸檔也要在 commit 前，產生的檔案才會被一併提交。

**為什麼要護欄**：push 動的是共享狀態；某些檔案（本地環境設定）容易誤 commit；review findings 不能盲目全套用。這些是人工每次重跑會漏掉的地方。

| skill | 作用 |
|---|---|
| `finalize` | 上述編排的入口 |
| `code-quality` | 簡化、命名、結構、清理 —— 機械式品質 |
| `code-correctness` | 會造成錯誤行為的 bug，以及「共用層已經有了卻又寫一份」 |
| `react-perf-check` | useEffect 依賴、Context value 穩定性、循環依賴、memo 誤用 |
| `commit` | 檔案範圍控制與留意檔案警告 |

### 獨立工具

| skill | 作用 |
|---|---|
| `create-migration` | 資料庫 migration，同時產生 MySQL／PostgreSQL／SQL Server 三種語法 |
| `doc` | 三階段共筆流程（蒐集脈絡 → 逐段精煉 → 讀者測試） |

`create-migration` 的使用頻率不高，但它封裝了一條代價很高的規則：**破壞性 DDL（DROP／RENAME）不可提前交付、也不可綁在停用該欄位的同一次釋出**。客戶端的 SQL 由對方 DBA 在無法精準控制的時間點執行，且通常早於程式上版 —— 對「新增」類安全，對「移除」類順序剛好相反。這條規則來自一次線上全站報錯。

## notes/

從實際故障與設計討論整理的技術筆記。

| 筆記 | 主題 |
|---|---|
| `jvm-heap-oom-diagnosis` | 從「猜哪個快取漏了」收斂到物件層級證據：histogram 判讀、jhsdb 採證 |
| `jpa-join-cartesian-product-first-vs-second-level` | 第一層 vs 第二層 JOIN 的笛卡兒積為何規模不同 |
| `Transaction-REQUIRES_NEW-Pattern` | 交易傳播與 Hibernate 判等契約 |
| `spring-transaction-rollback-exception-masking` | rollback 失敗會覆蓋原始例外，以及 Spring 留在哪裡的底 |
| `mssql-jdbc-12x-ssl-hikaricp-connection-pool-masking` | driver 預設值變更 ＋ 連線池造成的雙重症狀遮蔽 |
| `cli-java-test-execution-pitfalls` | surefire 不下探 `@Nested` 的假綠燈：`BUILD SUCCESS` 但測試沒跑 |
| `Consistency-Refactoring-Dialectics` | 對齊式重構何時該做：動機 × 對照組 × 時機三個維度 |

## 安裝

```sh
for s in skills/*/; do ln -sfn "$(pwd)/${s%/}" ~/.claude/skills/; done
```

`finalize` 的歸檔階段需要 OpenSpec plugin，沒有裝會跳過該階段。

## 關於這些內容

skill 與筆記都是工作中實際使用的版本，不是為了展示而寫的。公開版做過去識別化，
但流程、判斷邏輯與踩坑紀錄與原版一致。

`finalize` 在實際專案中被調用 41 次、跨 4 個專案；`code-quality` 27 次、
`code-correctness` 19 次、`react-perf-check` 23 次 —— 這幾個數字是它被設計成
sub-agent 編排而非單一檢查清單的原因：跑得夠多，才會發現自己 review 自己是無效的。
