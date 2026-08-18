---
title: "為何 CLI / Claude bash 跑的測試會出錯：JDK 來源與 surefire @Nested 兩個陷阱"
type: lesson
date: 2026-06-30
tags: [testing, maven, surefire, junit5, jdk, sdkman, non-interactive-shell, claude-code]
related: []
---

同一批核心模組的測試，在 IntelliJ 跑一直正常，改由 Claude 的 Bash 工具（非互動式 shell + maven CLI）跑卻「出錯」。事後查清：**兩個「錯」都不是程式碼 bug**——但性質不同：根因 A（Lombok）是環境造成的**假象**（測試其實能跑）；根因 B（@Nested）是下錯 CLI 過濾參數導致測試**真的沒被執行**（不是假象，是真漏跑）。本文記錄發生 → 診斷 → 解法 → 見解，避免未來脫離 IntelliJ 改走 terminal 時重踩。

> shell 啟動時機與 process 環境繼承的通用機制不在本文重述，只引用其結論。

---

## 背景

核心模組（Spring Boot 2.7.4、JUnit 5、maven-surefire 3.0.0-M5）的單元測試在 IntelliJ 點 Run 一切正常。某次改由 Claude Code 的 Bash 工具在命令列跑 `./mvnw test`，連續冒出兩個看似嚴重的問題。

## 問題

1. **看似 Lombok 編譯錯誤**：測試還沒跑就被編譯階段的 Lombok 相關錯誤擋下。
2. **測試「靜默沒跑」**：`mvn test -Dtest=HeadcountApprovalScheduleServiceTest` 只報 `Tests run: 2`。但這個測試類別其實有 **63 個 `@Test`**——只有 **2 個寫在頂層**，其餘 **61 個分散在 17 個 `@Nested` 內部類別**（例如 `Linearize` 這個 nested class 就有 11 個）。`-Dtest` 只報到那 2 個，讓人以為 `@Nested` 探索失效、61 個被靜默略過。

兩個現象都只在命令列出現，IntelliJ 從來沒事。

## 原因分析

### 根因 A — JDK 來源不對（非程式碼 bug）

Claude 的 Bash 是**非互動式 shell**，不會觸發 SDKMAN 的 auto-env，所以 PATH 上的 `java` 是系統預設的 **Java 25**，而非專案 `.sdkmanrc` 宣告的 `8.0.462-zulu`。

關鍵：**Lombok 靠掛鉤 javac 的內部 API（`com.sun.tools.javac.*`）運作，這些是非公開 API、會隨 JDK 改版變動。** Spring Boot 2.7.4 鎖定的 **Lombok 1.18.24（2022-04-18）官方支援上限就是 JDK 18**（該版 changelog：`JDK18 support added`），遠低於 Java 25，所以在 Java 25 上 annotation processor 直接炸 → 表現成「Lombok 編譯錯誤」。（逐版支援對照見 Lombok 官方 changelog。）

用 Java 8 跑 `test-compile` → `exit 0`，證實這不是程式碼問題。

> **精確措辭**：不是「Spring Boot + Lombok 只能用 Java 8」。真正的限制是「鎖定的舊 Lombok（1.18.24）官方支援到 **JDK 18** 為止，撐不到更新的 JDK」；Java 8 是這專案**選定的基準**（`.sdkmanrc` + pom `<java.version>1.8</java.version>`），介於下限（基準）與上限（Lombok 撐得住的最高 JDK）之間都可編。pom 的 `1.8` 是 **bytecode target / source level**（compiler flag，決定產出哪個版本的 .class），與這次的編譯失敗無關——因為**即使 target 設 1.8，實際執行編譯的 javac 仍是 Java 25 那一支**；崩在 Lombok 的是「跑 javac 的 JDK」，不是「產出的 bytecode 版本」。這是兩個獨立維度。要對齊的是「能正常編譯的 JDK 工具鏈」。想用新 JDK 就升 Lombok，不必被 8 綁死——升級對照：JDK19 → 1.18.26、JDK20 → 1.18.28、JDK21 → 1.18.30、JDK22 → 1.18.32（更新的 JDK 需再更新的 Lombok，見 Lombok 官方 changelog）。

### 根因 B — surefire 的 `-Dtest=類別名` 不下探 `@Nested`

測試 import 全是 JUnit 5 jupiter（無 JUnit 4 混用），surefire 也支援 `@Nested`。真正原因是：**`-Dtest=ClassName` 的類別過濾不會下探 `@Nested` 內部類別**（其 FQN 是 `Outer$Nested`），只比對到外層、只報頂層 flat `@Test` 的數量。

實證（全檔 63 個 `@Test`、17 個 `@Nested`）：

| 指令 | 結果 |
|------|------|
| `-Dtest=HeadcountApprovalScheduleServiceTest` | `Tests run: 2`（只有頂層 2 個 flat） |
| `-Dtest=HeadcountApprovalScheduleServiceTest*`（wildcard） | 仍 `2` |
| `-Dtest='HeadcountApprovalScheduleServiceTest$Linearize'`（明確 nested selector） | **`Tests run: 11`，全過** ✅ |

→ `@Nested` 沒壞，是**過濾參數**把它們濾掉了。完整 `mvn test`（不加 `-Dtest`）時 JUnit Platform 會正常探索所有 `@Nested`。

### 為何 IntelliJ 一直正常

IntelliJ 同時避開了這兩個**命令列特有**的陷阱：

- **避開 A**：run configuration 的 JRE 取 Project/Module SDK（= Java 8），與終端機 PATH 上的 `java` 無關。
- **避開 B**：直接用 JUnit Platform launcher 跑測試，原生探索 `@Nested`，不經過 surefire 的 `-Dtest` 過濾。

## 解決方式

| 問題 | 命令列正解 |
|------|-----------|
| JDK 來源（根因 A） | 跑 maven 前加 `source ~/.sdkman/bin/sdkman-init.sh && sdk env`——`sdk env` 讀 `.sdkmanrc`、不寫死路徑，讓非互動 shell 也對齊到專案宣告的 JDK |
| `@Nested` 沒跑（根因 B） | 用 `-Dtest='Class$Nested'`；或跑完整 `mvn test`（不加 `-Dtest` 過濾）。**不要**用 `-Dtest=純類別名` 然後相信它的數字 |

組合範例：

```bash
cd modules/core
source ~/.sdkman/bin/sdkman-init.sh && sdk env
./mvnw test -Dtest='HeadcountApprovalScheduleServiceTest$Linearize'
```

**持久化建議**（讓人與 Claude 都不靠肌肉記憶）：
- 在核心模組的 `CLAUDE.md` 加一條規則：「該模組的 `mvn`/`mvnw` 指令一律前綴 `source sdkman-init && sdk env`，以 `.sdkmanrc` 的 JDK 執行」。
- 更強的免記憶選項：Maven Toolchains（pom + `~/.m2/toolchains.xml`），讓 maven 不管被誰用什麼 java 啟動都強制用指定 JDK 工具鏈——但需動 pom 設定，較重。
- 互動式終端機只要開 SDKMAN auto-env（`sdkman_auto_env=true`），`cd` 進專案就自動切，連 `sdk env` 都免。

> 一條過時的 memory（`surefire-nested-gap`）原本誤診為「@Nested 探索失效、測試一律改 flat」，已更正為上述 `-Dtest` 過濾原因。

## 教訓

- **「IDE 沒事、terminal 出事」往往不是程式碼問題**，而是兩個層面的差異：**執行環境（JDK 從哪來）** + **測試啟動方式（IDE 的 JUnit Platform launcher vs maven surefire 的過濾）**。先查這兩者，別急著改 code。
- **非互動式 shell（CI、Claude 的 Bash、cron）不繼承互動式 shell 的環境切換**（SDKMAN auto-env、`.profile` 的 PATH 調整）。這是「在我機器上好好的」最常見的根源之一。
- **綠燈不等於有跑**：`-Dtest=類別名` 可以 `BUILD SUCCESS` 卻漏掉所有 `@Nested`。永遠看 `Tests run: N` 的實際數字，並對照你預期的測試數——數字對不上就是過濾或探索出問題，不是「沒有測試」。
- **版本相容性看的是「上限」而非「唯一解」**：舊工具（Lombok）撐不到太新的 runtime（JDK），但有一段相容區間；別把「基準版本」誤當成「唯一可用版本」。

