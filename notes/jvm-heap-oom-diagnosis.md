---
title: JVM heap OOM 診斷方法論（histogram 判讀、jhsdb 採證、in-heap vs 資料量驅動）
type: troubleshooting
date: 2026-07-13
tags: [jvm, oom, heap, hibernate, diagnosis, windows, procrun]
related: [jpa-join-cartesian-product-first-vs-second-level]
---

長時間運行的 Java 服務反覆 `OutOfMemoryError: Java heap space` 時，如何從「猜哪個快取漏了」收斂到「物件層級證據坐實兇手」的一套方法。整理自一次線上 OOM 事故的實戰。

## 術語（先對齊語彙）

| 術語 | 含義 |
|------|------|
| **heap / `-Xmx`** | JVM 存放執行期物件的記憶體區；`-Xmx` 是其上限。 |
| **`Java heap space` / `GC overhead limit exceeded`** | 兩種 OOM 變體，都代表「heap 被活著的物件塞爆、GC 收不回」。後者是 GC >98% 時間卻 <2% 回收。 |
| **retained / 可達性** | 物件只要還被活的引用鏈連著，GC 就不能回收。**洩漏 = 該回收的物件被引用鏈釘住。** |
| **heap histogram** | heap 的「物件普查表」：每類別幾個實例、共佔多少 bytes。**看得出被『什麼類別』塞滿，看不出『被誰持有』。** |
| **heap dump（`.hprof`）** | heap 完整快照。Eclipse MAT 的 **Dominator Tree** 能算出「誰 retain 最多」→ 指出持有者。檔案 ≈ heap 大小。 |
| **Hibernate session / persistence context / flush** | 一次交易內 Hibernate 追蹤所有載入 entity 的結構；flush（commit 前同步 DB）要走訪全部被追蹤的 entity 與集合。 |
| **lazy collection / `PersistentBag` / `CollectionEntry` / `IdentityMap`** | entity 的每個一對多集合，Hibernate 都建一個 `PersistentBag` 並在 session 登記一筆 `CollectionEntry`（`IdentityMap` 索引之）。**一個 entity 有 N 個集合 → 載入時就產生 N 份追蹤物件，即使集合是空的。** |

## 兩種成長模式（診斷方向完全不同）

| | **in-heap 快取型** | **資料量驅動型** |
|---|---|---|
| 機制 | 程式在記憶體累積 Map/List | 記憶體用量隨 DB 某表變大 |
| 重啟 | **歸零**（reset-on-restart） | **不歸零**（資料在 DB） |
| 典型 | 無界 key 的 `ConcurrentHashMap`、未淘汰快取 | 一次 `findAll` 全表物化、load-all 端點 |

**判斷關鍵**：若「重啟後更快 OOM、但請求速率沒變」→ 高度懷疑**非** in-heap 快取（它們重啟歸零，若填充速度只由請求量決定、速率又沒變，就無法解釋加速）→ 指向資料量驅動（DB 隨時間變大、重啟不清、越過「一次載入 ≈ heap」死線）。
> 前提：這條假設「快取填充速度只由請求**速率**決定」。若重啟後請求**組合（打到哪些端點）**變了，某個快取也可能填更快——所以這是**收斂方向的似真推理**，仍需 histogram 坐實，不是鐵證。

## 診斷流程

1. **確認 OOM 型別**：`Java heap space` / `GC overhead` → heap 問題（非 Metaspace、非 native）。
2. **靜態掃無界結構**：找 long-lived bean 的 static/instance 集合欄位，只 put 無淘汰、key 為請求衍生高基數值者。但**靜態只能找「可疑」，不能坐實**。
3. **用執行期證據坐實**：這一步不可省——沒有它，調查會停在「某快取可能是兇手」的錯誤結論。**先抓 histogram**（快、輕）：若頂端類別已直指兇手（如某 entity + 大量 Hibernate 追蹤物件）就夠；**唯有要確認「被誰持有」（持有鏈）時才升級到 dump** + MAT Dominator Tree（貴、檔案大）。
4. **判讀**（見下）。

## histogram 判讀（頂端類別 → 兇手方向）

| histogram 長相 | 判定 |
|---|---|
| 千萬級 `String` / `HashMap$Node`、instances 極多 | in-heap 快取容器（無界 Map） |
| 某業務 entity（如 `Candidate`）幾十萬～百萬 + 大量 `PersistentBag`/`CollectionEntry`/`IdentityMap` | **一次 `findAll` 全表物化 + Hibernate 集合追蹤**（資料量驅動） |
| `byte[]`/`int[]` 幾筆就佔幾百 MB（instances 少 bytes 巨） | 大物件（圖片/報表/文件處理） |
| `Object[]`/`String`/Model 混合、無單一霸主、或整表很稀 | 瞬時分配風暴 / 正常工作集撐爆 → 考慮調大 `-Xmx` |

**數量 vs 大小的形狀**：幾百萬個小物件 = 累積型；幾十個巨物件 = 分配型。histogram 給形狀，**持有鏈要靠 dump + Dominator Tree**。

## 陷阱

- **紅鯡魚（red herring）**：OOM 錯誤**冒出的位置**通常是「當下剛好在配置記憶體的倒楣操作」（背景 thread、無關端點），**很少是真正的洩漏源**。以 histogram/dump 定案，不要單信 stack trace 冒出點。
  - **例外**：若 OOM 剛好發生在**真兇正在大量配置的那一步**（如全表載入的 flush），該 request thread 的 stack 就確實指向兇手——但仍要 histogram 佐證，別單靠它。該次事故兩種都遇到：第一輪 stack 剛好命中 `findAll` 的 flush、本地重現卻冒在無關的背景 thread。
- **「加速」不等於「漏速變快」**：27 天→2 天可能只是「資料量越過死線的時機差」，不是漏速變化。
- **無聲成長**：某些累積路徑不寫 log（如條件檢查提早 return 前的 put），log 零活動**不能**證否，只是沒有正面證據。

## 現場採證（Windows / procrun / Java 11）

該服務的 prod 環境跑在 **Windows + Apache Commons Daemon（procrun / `prunsrv.exe`）+ Java 11**，服務帳號常為 LOCAL SERVICE。

### 工具是什麼

| 工具 | 是什麼 | 在採證裡做什麼 |
|------|--------|----------------|
| **jmap** | JDK 內建的 heap 分析工具（memory map） | 對目標 JVM 要 histogram 或 heap dump。**靠 attach 機制**：請 JVM 內部的 attach listener 執行緒配合，所以需要 JVM 還醒著、且執行者與 JVM **同帳號**。 |
| **jhsdb** | JDK 內建除錯器（HotSpot Debugger），Java 9+ 取代舊 `jsadb`/`jstack -F` 等 | 以子指令 `jhsdb jmap …` 做和 jmap 一樣的事，但走 **SA（Serviceability Agent）**：**從外部直接讀程序記憶體**，不靠 attach listener → 能讀卡死或跨帳號的 JVM。 |
| **SA（Serviceability Agent）** | JVM 的一套「從外部解讀另一個 JVM 記憶體結構」的機制，非獨立指令 | jhsdb 背後用的技術；理解成「不打擾目標 JVM、直接把它的記憶體當資料讀」。 |
| **jps** | JDK 內建的 Java 程序清單工具（JVM Process Status） | `jps -lv` 列出本機 JVM 的 PID 與啟動參數（含 `-Xmx`）。LOCAL SERVICE 起的服務常列不出來。 |
| **jstat / jcmd** | 其他 JDK 診斷工具（GC 統計 / 通用診斷命令），本案未用 | 補充：`jcmd <PID> GC.class_histogram` 也能出 histogram，同樣走 attach，取捨等同 jmap。 |
| **netstat** | 作業系統網路連線工具（非 JDK） | `netstat -ano` 用「埠 → PID」反查是哪個 JVM，多 Tomcat 並存時交叉鎖定。 |
| **Tomcat9w.exe** | procrun 的服務設定 GUI | 看／改該 Windows 服務的 JVM 參數（含 `-Xmx`），jps 看不到時的退路。 |

> 一句話取捨：**能 attach（醒著、同帳號）就用 `jmap`（最輕）；卡死或跨帳號就用 `jhsdb`（走 SA、繞過 attach）。**

### 採證指令

| 目的 | 指令 | 何時用／取捨 |
|------|------|------------|
| 抓 histogram（**JVM 自產**，最輕） | `jmap -histo <PID>` | 靠目標 JVM 內部 **attach listener 執行緒**回應；JVM 卡死、或與服務**不同帳號**會 attach 失敗 |
| 抓 histogram（**force**） | `jmap -F -histo <PID>` | `-F`＝對無回應 JVM 強制附掛，**只有 Java 8 有**；Java 11 已移除、本案用不到（提醒別照抄 Java 8 教學） |
| 抓 histogram（**SA**，從外部讀程序記憶體） | `jhsdb jmap --histo --pid <PID>` | SA＝Serviceability Agent，**不靠 attach listener、不受帳號限制、能讀卡死 JVM**——Java 11 跨帳號（LOCAL SERVICE）採證主力 |
| 抓 heap dump | `jmap -dump:format=b,file=… <PID>` | 比 SA `--binaryheap` 可靠；檔案 ≈ heap 大小、含個資 |
| 找 PID | `netstat -ano \| findstr :<AJP或HTTP埠>`（或 log Spring 行 `INFO <PID>`） | 多 Tomcat 時用 netstat 交叉鎖定 |
| 讀 `-Xmx` | `jps -lv`（常看不到 LOCAL SERVICE 的 JVM）→ 改看 `Tomcat9w.exe` Java 分頁 | |
| 預防：下次自動留證 | `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=…` | procrun 啟動參數加上 |
| 預防：不殭屍化 | `-XX:+ExitOnOutOfMemoryError` | OOM 立即退出＋服務自動重啟 |

### 常用旗標與變體

| 想做的事 | 寫法 | 說明 |
|----------|------|------|
| **輸出到檔案** | `jmap -histo <PID> > histo.txt` | histogram 是純文字，直接 stdout 重導向即可（jhsdb 版同理）。當初調查就是這樣存下來的。 |
| **只算存活物件** | `jmap -histo:live <PID>` | 加 `:live` 會**先觸發一次 Full GC** 再統計，濾掉待回收垃圾——查洩漏更乾淨，但正卡 OOM 邊緣的 prod 那次 GC 可能讓它更喘，故當初用**不帶** `:live` 的版本。 |
| **dump 只留存活物件** | `jmap -dump:live,format=b,file=heap.hprof <PID>` | 同樣 `live` 先 Full GC；`format=b`＝binary（給 MAT 讀），省略則為舊式文字格式。 |
| **dump 全部（含垃圾）** | `jmap -dump:format=b,file=heap.hprof <PID>` | 不觸發 GC，連待回收物件一起抓；想看「GC 前的完整現場」用這個。 |
| **jhsdb 對應版** | `jhsdb jmap --histo --pid <PID>`／`--binaryheap --dumpfile=heap.hprof` | 走 SA、繞過 attach（見上表）；`--binaryheap` 即 dump，實務上不如 `jmap -dump` 可靠。 |

> `:live` / `dump:live` 的共通代價：**都會 Full GC**。查洩漏（想確認「回收後還在的才是真兇」）時值得；prod 快撐不住時避免。

### 現場要點（非指令）

- **procrun OOM 後不自動 kill**，JVM 卡在原地苟延 → **爆掉當下仍可 attach，先採證再重啟**。
- **實戰順序**：跨帳號先直接上 **jhsdb**；若 SA 讀到壞頁 `readvirtual failed (0x8007001E)` → 退回 **`jmap -histo`**（賭 attach listener 還活著，能通就代表可用）。
- **Git Bash 坑**：參數用 `//svc`、路徑用正斜線、提權開啟。
- **hprof 含個資**（記憶體快照含姓名/email/履歷）：安全管道傳、用後刪。

## 相關

- [JPA 多表聯集的笛卡兒積](jpa-join-cartesian-product-first-vs-second-level.md) — 另一種 Hibernate 記憶體放大（JOIN 笛卡兒積）
