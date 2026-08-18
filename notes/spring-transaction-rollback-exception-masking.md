---
title: Spring 交易 rollback 失敗遮蔽原始例外
type: lesson
date: 2026-05-29
project:
tags: [spring, transaction, exception, hikaricp, error-reporting]
related: ["[mssql-jdbc SSL 與連線池遮蔽](mssql-jdbc-12x-ssl-hikaricp-connection-pool-masking.md)", "[REQUIRES_NEW 交易傳播](Transaction-REQUIRES_NEW-Pattern.md)"]
---

## 背景

104 履歷匯入 job（`ImportResumeFrom104EmailJob` → `ImportResumeService.importResumeFrom104ByEmail`）採「每筆履歷各開一條 `REQUIRES_NEW` 交易」（`ImportResumeTransactionHelper.processOneResume`）的架構，外層迴圈以 try/catch 隔離單筆失敗，並對失敗履歷寄通知信給系統管理員。

某次管理員收到的通知信內容為：

```
TransactionSystemException: Could not roll back JPA transaction
  └─ caused by: TransactionException: Unable to rollback against JDBC Connection
       └─ caused by: SQLException: Connection is closed
```

收到信的人完全無法判斷：這是暫時性連線問題、資料問題、還是業務邏輯錯誤？

## 問題

這串 stack trace **不含真正的失敗原因**。它是一個「次要例外」蓋掉了「主要例外」：

1. 交易進行中，JDBC 物理連線被中斷（DB / 網路暫時性問題）。
2. 某條 SQL 因連線已死而拋出**原始例外**（真兇）。
3. Helper 的 catch 把它包成 `throw new RuntimeException(e)` re-throw——Spring 宣告式交易預設只對 unchecked exception 回滾，包成 `RuntimeException` 是為了觸發 `REQUIRES_NEW` 的 rollback（每筆履歷各開一條獨立交易，見 [REQUIRES_NEW 交易傳播](Transaction-REQUIRES_NEW-Pattern.md)）。
4. Spring 嘗試 rollback，卻發現連線已關閉 → 拋出 `TransactionSystemException`。
5. 這個「rollback 失敗」的例外往上傳，外層 catch 直接把它的 stack trace 寫進通知信內容（`e.printStackTrace` 到信件用的 `StringWriter`）。

於是真兇從未出現在信裡——信只說「我想復原交易但連線已斷」，卻沒說「當初為什麼失敗」。

## 原因分析

關鍵在於：**原始例外其實沒有遺失，只是沒被去拿。**

Spring 的 `AbstractPlatformTransactionManager.completeTransactionAfterThrowing` 在 rollback 本身丟出 `TransactionSystemException` 時，會這樣處理（Spring 5.3.x / Spring Boot 2.7.4）：

```java
catch (TransactionSystemException ex2) {
    logger.error("Application exception overridden by rollback exception", ex);
    ex2.initApplicationException(ex);   // ← 把原始例外存進獨立欄位
    throw ex2;
}
```

所以原始例外被掛在 `TransactionSystemException.getApplicationException()`，而 server log 也會出現一行 `Application exception overridden by rollback exception`（含原始例外）。

**陷阱：不能用 `NestedExceptionUtils.getMostSpecificCause()` 取根因。**
原始例外存在 `applicationException` 這個**獨立欄位**，不在 `cause` 鏈上。`getMostSpecificCause()` 只沿 cause 鏈走，對 `TransactionSystemException` 只會回到 rollback 失敗的 cause（仍是「Connection is closed」），抓錯對象。要取原始例外，必須明確 `instanceof TransactionSystemException` 後呼叫 `getApplicationException()`。

## 解決方式

在錯誤回報層（組裝通知信的地方）解遮，而非在 Helper 層攔截——因為 Helper 必須 re-throw 才能觸發 `REQUIRES_NEW` 的 rollback，無法阻止 rollback 失敗。

核心模組 `ImportResumeService.buildResumeImportErrorReport(Throwable)` 的邏輯：

```java
Throwable reportable = e;
boolean rollbackMasked = false;

if (e instanceof TransactionSystemException) {
    Throwable appEx = ((TransactionSystemException) e).getApplicationException();
    if (appEx != null) {
        reportable = appEx;
        rollbackMasked = true;
        // 剝除 Helper re-throw 包覆用的 RuntimeException wrapper（取不到 cause 時退回 wrapper 本身，避免 NPE）
        // 用 getClass() == RuntimeException.class 精確比對而非 instanceof，只剝「恰好是原生 RuntimeException」的純包裝殼，
        // 不誤剝本身具診斷意義的 RuntimeException 子類（如各種業務例外）
        if (reportable.getClass() == RuntimeException.class && reportable.getCause() != null) {
            reportable = reportable.getCause();
        }
    }
}
// 印 reportable 的完整 stack 作為信件主體；rollbackMasked 時於末尾附註「rollback 亦失敗、連線已關閉」
```

三個設計重點：

- **優先揭露原始根因**：信件主體換成 `getApplicationException()`，而非最外層的 rollback 失敗例外。
- **保留 rollback 失敗作為附註**：「rollback 亦失敗、連線已關閉」本身是有價值的訊號（暗示連線中斷類問題），不丟棄，只是不當主角。
- **零回歸**：非 `TransactionSystemException`、或 `getApplicationException()` 為 null 時，退回原本的 `printStackTrace` 行為。

## 教訓

- **rollback 失敗會「覆蓋」原始例外，但 Spring 有幫你留底**：看到 `Could not roll back` / `Unable to rollback` 時，先去 `TransactionSystemException.getApplicationException()` 找真兇，並搜 server log 的 `Application exception overridden by rollback exception`。
- **`getMostSpecificCause()` 不是萬靈丹**：當「真正的原因」掛在獨立欄位而非 cause 鏈上時（如 `applicationException`、HikariCP 的 connection-is-closed 也常見類似遮蔽，見 [mssql-jdbc SSL 與連線池遮蔽](mssql-jdbc-12x-ssl-hikaricp-connection-pool-masking.md)），沿 cause 鏈走會抓錯。
- **解遮要做在「呈現/回報層」**：例外被包裝是為了控制流程（觸發 rollback），但人在讀的訊息應還原到最有診斷價值的那一層。包裝與呈現是兩件事。
- **次要失敗不該吃掉主要失敗的資訊**：任何「在錯誤處理中又發生錯誤」的場景（rollback 失敗、cleanup 失敗、close 失敗），都要刻意保留原始錯誤，否則會把可診斷問題變成不可診斷問題。

## 相關

- [mssql-jdbc SSL 與連線池遮蔽](mssql-jdbc-12x-ssl-hikaricp-connection-pool-masking.md) — driver 預設值變更引發的同型遮蔽
- [REQUIRES_NEW 交易傳播](Transaction-REQUIRES_NEW-Pattern.md) — 本案的交易切分前提
