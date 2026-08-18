---
name: create-migration
description: 建立資料庫 migration SQL，自動產生 MySQL/MariaDB/H2、PostgreSQL、SQL Server 三種語法。當需要新增資料表、欄位、修改欄位、INSERT 資料等資料庫變更時觸發。
argument-hint: "[操作描述，例如：在 candidate 表新增 phone 欄位]"
---

# 建立資料庫 Migration SQL

根據 **$ARGUMENTS** 產生三種資料庫語法的 migration SQL。

## 檔案位置

所有 migration 檔案放在核心模組下：

```
modules/core/docs/database/migrations/
```

## 執行步驟

### 1. 釐清需求

從 $ARGUMENTS 或對話脈絡中判斷：

- **操作類型**：CREATE TABLE / ADD COLUMN / ALTER COLUMN / INSERT / UPDATE / DROP 等
- **目標表**：哪張資料表
- **欄位細節**：欄位名稱、型別、約束

如果資訊不足，向用戶確認。

### 2. 查詢 Entity 定義（如適用）

如果操作涉及既有資料表：

1. 搜尋對應的 JPA Entity 類別（在 `src/main/java/io/talentonline/tas/entity/` 下）
2. 確認 `@Column(name = "...")` 註解 — **欄位名稱必須以 `@Column` 為唯一依據**
3. 確認欄位型別（Java → SQL 對應）

### 3. 決定檔案歸屬

| 變更類型 | 目標檔案 |
|---------|---------|
| 功能相關的 DDL（CREATE TABLE / ALTER 等） | `YYYY-MM-DD_功能描述.sql`（新建） |
| 功能相關的 code / letter_template 新增 | **寫在該功能的 `YYYY-MM-DD_功能描述.sql` 內**（與 schema 一起） |
| 獨立的 code 新增（無對應 feature SQL） | `code-additions.sql`（按時間升序 append 到末尾） |
| 獨立的 letter_template 新增（無對應 feature SQL） | `letter-template-additions.sql`（按時間升序 append 到末尾） |

**分流原則**：
- 優先判斷是否有（或即將建立）對應的 feature SQL — 有就寫在 feature SQL，不要在 `*-additions.sql` 留參照記錄
- 只有「通用補充、與任何功能 table 無關」的 code / letter_template 才進 `*-additions.sql`

**新增 code 時必須同步更新**：
- `src/main/resources/data/Code.json`（用於專案初始化；DataInitializer 在 code 表為空時讀取此檔建立資料）
- SQL 檔案（用於現有環境更新）

**新增 letter_template 時必須同步更新**：
- `src/main/resources/data/LetterTemplate.json`（用於專案初始化）
- SQL 檔案（用於現有環境更新）

### 4. 產生 SQL

**必須產生三個區段**，依以下順序排列：

```
1. MySQL / MariaDB / H2（預設啟用，不包在註解中）
2. PostgreSQL（用 /* */ 註解包裹）
3. SQL Server（用 /* */ 註解包裹）
```

#### 型別對應表

| MySQL / MariaDB / H2 | PostgreSQL | SQL Server |
|----------------------|------------|------------|
| `VARCHAR(n)` | `VARCHAR(n)` | `NVARCHAR(n)` |
| `TEXT` | `TEXT` | `NVARCHAR(MAX)` |
| `LONGTEXT` | `TEXT` | `NVARCHAR(MAX)` |
| `INT` / `INTEGER` | `INT` / `INTEGER` | `INT` |
| `DATETIME` | `TIMESTAMP` | `DATETIME2` |
| `DATETIME(6)` | `TIMESTAMP` | `DATETIME2` |
| `BIT` / `BIT(1)` | `BOOLEAN` | `BIT` |
| `DEFAULT 0`（BIT） | `DEFAULT FALSE` | `DEFAULT 0` |
| `DEFAULT 1`（BIT） | `DEFAULT TRUE` | `DEFAULT 1` |

#### UUID 產生函數

| 資料庫 | UUID 函數 |
|--------|----------|
| MySQL / MariaDB / H2 | `UUID()` |
| PostgreSQL | `md5(random()::text \|\| clock_timestamp()::text)::uuid` |
| SQL Server | `NEWID()` |

#### 中文字串前綴

| 資料庫 | 前綴 |
|--------|------|
| MySQL / MariaDB / H2 | 不需要 |
| PostgreSQL | 不需要 |
| SQL Server | `N'...'` |

#### 冪等性設計

| 操作 | MySQL / MariaDB / H2 | PostgreSQL | SQL Server |
|------|----------------------|------------|------------|
| CREATE TABLE | `CREATE TABLE IF NOT EXISTS` | `CREATE TABLE IF NOT EXISTS` | `IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='...' AND xtype='U')` |
| ADD COLUMN | `ADD COLUMN IF NOT EXISTS` 或 Stored Procedure | `ADD COLUMN IF NOT EXISTS` | `IF NOT EXISTS (SELECT * FROM sys.columns WHERE ...)` |
| INSERT | `WHERE NOT EXISTS (SELECT 1 ...)` | `WHERE NOT EXISTS (SELECT 1 ...)` | `WHERE NOT EXISTS (SELECT 1 ...)` |
| 條件邏輯 | `SET @var = ...; PREPARE stmt FROM @sql;` | `DO $$ BEGIN ... END $$;` | `IF ... BEGIN ... END` |

#### MySQL 專屬語法

- CREATE TABLE 加上 `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4`（僅在建立新表時）
- UNIQUE KEY 用 `UNIQUE KEY uk_表名_欄位 (欄位)`

#### PostgreSQL 專屬語法

- 不需要 ENGINE / CHARSET 子句
- UNIQUE 用 `UNIQUE (欄位)` 或 `CONSTRAINT uk_表名_欄位 UNIQUE (欄位)`
- 不需要 `GO` 分隔符

#### SQL Server 專屬語法

- 用 `GO` 分隔批次
- UNIQUE 用 `CONSTRAINT uk_表名_欄位 UNIQUE (欄位)`

### 5. 檔案格式

功能 Migration 檔案的標準格式：

```sql
-- ============================================================================
-- 功能：{功能中文名稱}
-- 日期：{YYYY-MM-DD}
-- 說明：{變更說明}
-- 冪等性：所有語句均可重複執行
-- ============================================================================

-- MySQL / MariaDB / H2 -------------------------------------------------------

{MySQL SQL 語句}

-- PostgreSQL -----------------------------------------------------------------

/*
{PostgreSQL SQL 語句}
*/

-- SQL Server -----------------------------------------------------------------

/*
{SQL Server SQL 語句}
*/


-- ============================================================================
-- 欄位說明
-- ============================================================================
-- {欄位名} : {說明}
-- ============================================================================
```

### 5a. Code / LetterTemplate INSERT 格式範例

#### Code INSERT 範例

```sql
-- MySQL / MariaDB / H2 -------------------------------------------------------
INSERT INTO code (uuid, type, value, text, source, priority)
SELECT UUID(), 'category', 'codeType', '代碼類型', 'system', 0
WHERE NOT EXISTS (
    SELECT 1 FROM code WHERE type = 'category' AND value = 'codeType'
);
GO

-- PostgreSQL -----------------------------------------------------------------
/*
INSERT INTO code (uuid, type, value, text, source, priority) VALUES (
    md5(random()::text || clock_timestamp()::text)::uuid, 'category', 'codeType', '代碼類型', 'system', 0
);
GO
*/

-- SQL Server -----------------------------------------------------------------
/*
INSERT INTO code (uuid, type, value, text, source, priority)
SELECT NEWID(), 'category', 'codeType', N'代碼類型', 'system', 0
WHERE NOT EXISTS (
    SELECT 1 FROM code WHERE type = 'category' AND value = 'codeType'
);
GO
*/
```

#### Code.json 同步格式

每筆 code（包括 category）必須同步 append 到 `src/main/resources/data/Code.json` 陣列末尾。**Category 項目不帶 `priority`**；非 category 項目依顯示順序遞增 priority（10, 20, 30, ...）。

```json
{
  "type": "category",
  "value": "codeType",
  "text": "代碼類型",
  "source": "system"
},
{
  "type": "codeType",
  "value": "value1",
  "text": "顯示文字",
  "source": "system",
  "priority": 10
}
```

注意：append 時，原本陣列最後一筆物件後需補逗號。Code.json 不需要 `uuid` 欄位（由 DataInitializer 在初始化時產生）。

#### LetterTemplate 欄位對應

| 欄位 | 說明 |
|------|------|
| `uuid` | UUID 函數產生 |
| `version` | 固定為 `0` |
| `code` | 模板代碼（對應 LetterTemplate enum） |
| `name` | 模板名稱 |
| `subject` | 信件主旨 |
| `content` | RTX 格式的 JSON 字串（支援 paragraph、numbered-list、bulleted-list） |
| `_usable_codes` | 可用代碼，以 `", "` 分隔並依字母排序 |

#### LetterTemplate INSERT 範例

```sql
-- MySQL / MariaDB / H2 -------------------------------------------------------
INSERT INTO letter_template (uuid, version, code, name, subject, content, _usable_codes)
SELECT UUID(), 0, 'templateCode', '模板名稱', '信件主旨',
       '{"type":"rtx","children":[...]}',
       '[code1], [code2], [code3]'
WHERE NOT EXISTS (
    SELECT 1 FROM letter_template WHERE code = 'templateCode'
);
GO
```

PostgreSQL 與 SQL Server 變體比照 Code INSERT 範例（UUID 函數、中文前綴規則相同）。

**LetterTemplate.json 同步格式**（`src/main/resources/data/LetterTemplate.json`）：

```json
{
  "code": "模板代碼",
  "name": "模板名稱",
  "subject": "信件主旨",
  "content": "RTX 格式的 JSON 字串",
  "usableCodes": ["[變數1]", "[變數2]"]
}
```

### 6. 驗證

產出後自檢：

- [ ] 三種資料庫語法都有
- [ ] 欄位名稱與 Entity `@Column` 一致
- [ ] 型別對應正確（參照型別對應表）
- [ ] 冪等性：可重複執行不出錯
- [ ] SQL Server 中文字串有 `N'...'` 前綴
- [ ] 檔案歸屬正確（feature SQL vs `*-additions.sql`，參照步驟 3）
- [ ] 寫入 `*-additions.sql` 時，append 到檔案末尾（時間升序）
- [ ] 新增 code 時，已同步更新 `Code.json`（category 不帶 priority）
- [ ] 新增 letter_template 時，已同步更新 `LetterTemplate.json`

### 7. 輸出摘要

完成後顯示：
- 建立或修改的檔案路徑
- 變更摘要（哪張表、什麼操作）
- 提醒：如果有新增 Entity 欄位，確認是否已加上 `@Column(name = "...")`
