---
title: mssql-jdbc 12.x SSL 憑證錯誤與 HikariCP 連線池遮蔽效應
type: lesson
date: 2026-04-09
tags: [mssql-jdbc, hikaricp, ssl, connection-pool, troubleshooting]
related: [spring-transaction-rollback-exception-masking]
---

## 背景

`mssql-jdbc` 從 10.x 版本開始，將 `encrypt` 屬性的預設值從 `false` 改為 `true`。這意味著所有新建的 SQL Server 連線都會強制進行 SSL 加密握手，並驗證伺服器端的憑證是否被 JVM truststore 信任。

在內網環境中，SQL Server 通常使用自簽名憑證，這些憑證不在 JVM 預設的 `cacerts` 中，因此升級驅動版本後會立即觸發 `PKIX path building failed` 錯誤。

## 問題現象

升級 `mssql-jdbc` 至 12.x 後，應用程式中出現兩種不同表現：

- **DriverManager 直連**：每次操作都失敗，錯誤訊息明確，問題立即被發現
- **HikariCP 連線池**：平時正常運作，僅在極少數情況下偶發一次 SSL 憑證錯誤

偶發錯誤的 log 特徵：

```
SQLServerException: "encrypt" property is set to "true" and
"trustServerCertificate" property is set to "false" but the driver
could not establish a secure connection to SQL Server by using
Secure Sockets Layer (SSL) encryption: Error: PKIX path building failed
```

這種「直連必定失敗、連線池幾乎不出錯」的現象，容易導致錯誤的因果推斷：以為改用連線池就能解決 SSL 問題。

## 原因分析

SSL 憑證驗證發生在 **TCP 連線建立後的 SSL 握手階段**，也就是只有在「建立新連線」時才會執行。一旦連線建立完成，後續透過該連線的所有 SQL 操作都不會再次觸發握手。

因此問題的本質是：**憑證信任失敗是一個持續存在的狀態**，但它只在建立新連線的瞬間被檢測到。任何減少新連線建立頻率的機制，都會降低問題的可見度。

## 連線池遮蔽機制

HikariCP 連線池在啟動時建好一批連線，後續的資料庫操作都是重用這些既有連線，不會重新進行 SSL 握手。以實際案例中的設定為例：

| 參數 | 值 | 效果 |
|------|-----|------|
| MinimumIdle | 30 | 池中常駐 30 條連線 |
| MaximumPoolSize | 30 | 最多 30 條，不會動態擴張 |
| MaxLifeTime | null | 連線不主動回收 |
| IdleTimeout | 600000 | 閒置 10 分鐘才回收 |

在這組設定下，連線幾乎不會被回收或重建。SSL 憑證驗證的問題「一直存在」，但因為沒有新連線被建立，所以不會被觸發。

> **Q：啟動時建立初始連線不也會 SSL 握手失敗嗎？**
> 會。但 HikariCP 預設有連線建立的重試機制，且如果應用在驅動升級前已經啟動、連線池已建好，重啟前都不會觸發。實務上這個問題常在「已運行的環境」中被忽略，直到下次重啟才暴露。

> **注意：MaxLifeTime=null 的隱患**
> 連線永不主動回收，意味著如果 SQL Server 端有 idle session timeout，連線會在 DB 端被切斷，但應用端不知情，下次使用時才會報錯。建議設定合理的 MaxLifeTime（如 30 分鐘），讓連線池主動輪換連線。

**偶發觸發的因果鏈：**

```
網路瞬斷 / SQL Server 端 idle session timeout / TCP keepalive 失敗
  → 既有連線斷開
    → HikariCP 偵測到並嘗試補建新連線
      → SSL 握手
        → 憑證驗證失敗
          → 報錯
```

偶發的頻率取決於外部因素導致連線斷開的頻率，而非問題本身是否存在。

相對地，`DriverManager.getConnection()` 每次呼叫都建立全新連線，每次都會進行 SSL 握手，因此問題會 100% 重現。

## 修復歷程與盲點

### 某客戶專案的修復過程（2026-03-06）

1. **發現問題**：升級 `mssql-jdbc` 12.x 後，HR 資料庫的 JDBC 直連全部失敗
2. **第一次修復**（`a907552`）：在三個 Service 的直連程式碼中加上 `trustServerCertificate=true` — 正確修復了直連的問題
3. **架構檢視**（`17f805b`）：發現系統中同時存在 HikariCP 與 DriverManager 兩套連線方式
4. **重構**（`722b8b1`）：將直連統一改為走 HikariCP，並在 HikariCP 設定中加入 `trustServerCertificate=true`

### 盲點

修復時的觀察是「直連會失敗，HikariCP 不會」，自然推導出「統一改用 HikariCP 就能解決」。這個推論的結果恰好是正確的（因為重構時也加上了 `trustServerCertificate=true`），但推論的因果關係是錯的。

真正解決問題的是 `trustServerCertificate=true`，不是「改用 HikariCP」。HikariCP 只是遮蔽了問題。

### 漏網之魚

主程式的 `ImportResumeFrom104EmailJob` 中有一段 DB 診斷程式碼，使用 `DriverManager.getConnection()` 直連，繞過 HikariCP。因為這段程式碼只在排程 job 發生異常時才執行，且主資料庫連線平時走 HikariCP 正常運作，所以直到一個月後（2026-04-09）才偶發觸發。

### 直連在診斷場景的合理性

這段直連程式碼的用途是在錯誤發生時執行 `sp_who2` 診斷 SQL Server 的 session 狀態。理論上，用獨立連線做診斷有其道理：當懷疑連線池本身有問題（如 connection leak、pool exhaustion）時，透過連線池拿連線來診斷連線池，結果可能不可靠。

但實務上，這段診斷程式碼本身因為 SSL 錯誤而失敗，反而無法完成診斷任務。而且 HikariCP 的 metrics 已經透過 `DataSourceProfiler` API 取得，`sp_who2` 查的是 SQL Server 端的 session 資訊，用連線池的連線一樣能查。因此最終改為使用已注入的 `dataSource` 取得連線。

## 解決方案

### 根本修復：信任自簽名憑證

在 JDBC URL 或連線設定中加入 `trustServerCertificate=true`，讓 SSL 握手跳過憑證鏈驗證：

**方式一：JDBC URL 參數**

```
jdbc:sqlserver://host:1433;databaseName=db;trustServerCertificate=true
```

**方式二：HikariCP data-source-properties**

```properties
spring.datasource.hikari.data-source-properties.trustServerCertificate=true
```

**方式三：完全關閉加密**（不建議）

```
jdbc:sqlserver://host:1433;databaseName=db;encrypt=false
```

### 消除直連

將散落在各處的 `DriverManager.getConnection()` 統一改為使用 Spring 管理的 `DataSource`，確保所有連線都繼承相同的 SSL 設定。

## 教訓

1. **連線池能遮蔽連線階段的問題**。任何只在建立新連線時觸發的錯誤（SSL、認證、網路），都可能被連線池的重用機制隱藏，形成「偶發」的假象。修復連線問題時，應直接用 `DriverManager` 驗證新連線是否能正常建立，而非依賴連線池的行為來判斷。

2. **「改 A 後問題消失」不等於「A 是根因」**。改用 HikariCP 後問題消失，但根因是 SSL 憑證信任，不是連線管理方式。修復後應驗證因果關係：將 `MaxLifeTime` 暫時調低（如 30 秒），強制連線池在短時間內輪換所有連線，觀察是否仍有 SSL 錯誤。如果有，代表根因未解決。

3. **JDBC 驅動的預設值變更是高風險的 Breaking Change**。`encrypt` 預設值從 `false` 改為 `true` 不會產生編譯錯誤，只會在運行時、特定環境下才暴露。升級 JDBC 驅動時應重點檢查連線屬性的預設值變更。

4. **排查完一處直連，搜尋全部直連**。程式碼中可能有多處使用 `DriverManager.getConnection()` 的地方，修復時應全域搜尋，避免漏網之魚在數週後才偶發觸發。

## 相關

- [Spring 交易 rollback 失敗遮蔽原始例外](spring-transaction-rollback-exception-masking.md) — 同一條連線池上的另一種例外遮蔽
