---
title: "Transaction REQUIRES_NEW Pattern：跨事務寫入的陷阱與解法"
type: lesson
date: 2026-03-20
project:
tags: [spring, transaction, hibernate, jpa, optimistic-locking]
related: ["[Spring 交易 rollback 失敗遮蔽原始例外](spring-transaction-rollback-exception-masking.md)"]
---

本文件記錄在學校別名自動學習功能中遇到的 `@Transactional(propagation = REQUIRES_NEW)` 問題，以及解決方式。適用於所有在主事務中需要獨立 commit 副作用的場景。

---

## 背景

學校篩選的 AI 模糊比對功能會在匹配成功時，自動將候選人使用的校名（如「台大」）新增為該學校的別名。這個「新增別名」操作需要獨立 commit，不受外層事務（如 `saveCriteria` 重算所有候選人分數）的成敗影響。

---

## Persistence Context 基礎

每個 `@Transactional` 方法執行時，Hibernate 維護一個 **persistence context**（一級快取）。核心規則：

- 同一個 primary key 在同一個 context 中只有**一個** Java 物件
- 任何對 managed entity 的欄位修改都會被 dirty-checking 追蹤
- 事務 commit 時，所有 dirty entity 會自動 flush（寫入 DB）

```
事務開始
  ↓
entityManager.find(School.class, "aaa")  → 載入 School@0x1234 (version=5)
  ↓
schoolRepository.findByCode("1057")      → 回傳同一個 School@0x1234（context 中已有）
  ↓
school.setAliases(...)                   → School@0x1234 被標記為 dirty
  ↓
事務 commit
  → dirty-checking 發現 School@0x1234 被修改
  → UPDATE school SET _aliases=..., version=6 WHERE uuid='aaa' AND version=5
```

---

## @Version 與 Optimistic Locking

School entity 有 `@Version private Integer version`。Hibernate 在 flush 時會加上 version 條件：

```sql
UPDATE school SET _aliases='台大', version=6
WHERE uuid='aaa' AND version=5
--                             ↑ 如果別的事務已改成 6，影響 0 行
--                               → OptimisticLockException
```

這個機制確保併發修改不會互相覆蓋，但在 `REQUIRES_NEW` 場景中會造成問題。

---

## 問題：兩個事務持有同一筆 Entity

### 錯誤的做法

```java
// SchoolAliasResolverService.resolve()
School school = schoolRepository.findByCode(schoolCode);  // ← 載入到外層 context
boolean added = schoolService.addAlias(schoolCode, alias); // ← REQUIRES_NEW 內層事務
return school;                                             // ← 回傳 managed entity
```

時序圖：

```
外層事務 (saveCriteria)
│
├── school = findByCode("1057")          ← School(version=5) 載入外層 context
│
├── addAlias("1057", "台大")             ← REQUIRES_NEW 開始
│   └── [內層事務]
│       ├── school2 = findByCode("1057") ← 新 context，載入 School(version=5)
│       ├── school2.setAliases(["台大"])
│       ├── save → UPDATE version=6      ← commit！DB version 現在是 6
│       └── [內層事務結束]
│
├── ... 繼續處理其他候選人（執行多個查詢）...
│
└── commit
    ↓
    Hibernate auto-flush 檢查外層 context:
    School(version=5) 在 context 中 →

    情況 A: entity 未被修改 → 不 flush → OK（但不保證）
    情況 B: auto-flush 被觸發 → flush School(version=5) 回 DB
            → 覆蓋內層事務寫入的 version=6 的資料 ✗
            → 或觸發 OptimisticLockException ✗
```

### Auto-flush 觸發條件

Hibernate `FlushMode.AUTO`（預設）會在以下時機自動 flush：

1. **執行 JPQL / Criteria 查詢前** — 確保查詢能看到 context 中的修改
2. **事務 commit 前** — 確保所有修改寫入 DB
3. **手動呼叫 `entityManager.flush()`**

在 `saveCriteria` 的候選人迴圈中，每次 `screen()` 都會執行多個查詢，任何查詢都可能觸發 auto-flush。

---

## 解法：Detached Snapshot

### 核心原則

> 在 `REQUIRES_NEW` 場景中，外層事務**絕不應持有**內層事務修改的同一筆 entity。

### 正確的做法

```java
// SchoolService（被 Spring proxy 代理，REQUIRES_NEW 生效）
@Transactional(propagation = Propagation.REQUIRES_NEW)
public School resolveAndAddAlias(String schoolCode, String alias) {
    School school = repository.findByCode(schoolCode);  // 內層 context 專屬
    if (school == null) return null;

    // 建立 detached snapshot — 不受任何 context 管理
    School snapshot = new School();
    snapshot.setUuid(school.getUuid());
    snapshot.setName(school.getName());
    snapshot.setCode(school.getCode());

    // 修改 + 保存（在內層事務中 commit）
    if (!school.getAliases().contains(alias)) {
        List<String> updated = new ArrayList<>(school.getAliases());
        updated.add(alias);
        school.setAliases(updated);
        repository.save(school);
    }

    return snapshot;  // 回傳 detached 物件
}

// SchoolAliasResolverService.resolve()
School school = schoolService.resolveAndAddAlias(schoolCode, alias);
// school 是 detached snapshot：
// - 不在任何 persistence context 中
// - 不會被 dirty-checking 追蹤
// - 不會觸發 auto-flush
// - 不會有 version 衝突
return school;
```

時序圖：

```
外層事務 (saveCriteria)
│
├── resolveAndAddAlias("1057", "台大")     ← REQUIRES_NEW
│   └── [內層事務]
│       ├── school = findByCode("1057")     ← 內層 context 專屬
│       ├── snapshot = new School(...)      ← 手動建立，不受管理
│       ├── school.setAliases(["台大"])
│       ├── save → commit                  ← 別名保存成功，version=6
│       └── return snapshot
│
├── school = snapshot                       ← 外層 context 中沒有 School entity
│                                             dirty-checking 不追蹤
│                                             auto-flush 不影響
│
├── criteriaContainsSchool(school.getUuid()) ← 只用 uuid 字串比對
│
└── commit                                  ← 外層 context 乾淨，無衝突 ✓
```

---

## 使用 REQUIRES_NEW 的注意事項

| 注意事項 | 說明 |
|---------|------|
| **Self-call 無效** | 同一個 class 內的方法呼叫不經過 Spring proxy，`REQUIRES_NEW` 不會生效。必須透過注入的**不同 bean** 呼叫 |
| **避免 entity 外洩** | 內層事務的 managed entity 不應回傳給外層，改用 detached snapshot 或只回傳必要的值（uuid、name） |
| **內層 rollback 不影響外層** | `REQUIRES_NEW` 的事務是獨立的，內層 rollback 不會導致外層 rollback（除非例外被拋出） |
| **外層 rollback 不影響內層** | 這正是使用 `REQUIRES_NEW` 的目的 — 副作用（如別名保存）需要獨立持久化 |
| **死鎖風險** | 外層持有某行的鎖，內層也嘗試鎖同一行 → 死鎖。用 detached snapshot 避免外層載入同一筆 entity |
| **判等契約** | 跨 context 後同一筆資料會有多個 Java 物件。任何依賴 `equals`／`hashCode` 的集合操作都必須確認 entity 已定義判等，詳見下節 |

---

## Entity 判等契約：跨 context 的另一面

上文「Persistence Context 基礎」的第一條規則是：

> 同一個 primary key 在同一個 context 中只有**一個** Java 物件

這條規則有個必須一併記住的**反面**：**一旦跨出 context，同一筆資料就會有多個物件**。而系統中有一批集合操作正是靠著「只有一個物件」在運作——它們的語意全部委託給元素的 `equals`／`hashCode`，未定義時退化為 identity 比較。

### 會出事的操作

| 操作 | 判等失效時的後果 |
|---|---|
| `Set.add`（元素對應有 PK 的 join table） | 同一筆資料被當成兩個元素 → 重複 INSERT → 撞主鍵、交易 rollback |
| `contains` / `remove` | 比不中 → 該移除的關聯留著（**靜默**） |
| 權限判斷中的 `contains` | 誤判為無權限 → 使用者看不到該看的資料（**靜默**） |
| `stream().distinct()` | 去重失效 → 重複處理（**靜默**） |
| `removeAll` / `retainAll` / 差集比對 | 新增／移除的判斷失準（**靜默**） |
| entity 當作 `Map` 的 key | 取不到值（**靜默**） |

只有第一列會報錯，其餘都是安靜地做錯事。

### 檢查順序

改動要把 entity 帶出它被載入的交易時（`REQUIRES_NEW`、`NOT_SUPPORTED`、`@Async`、事件監聽器、跨排程傳遞），**先看操作、再看 entity**：

1. 這段程式碼有沒有用到上表的任一操作？
2. 若有，涉及的 entity 是否已定義 `equals`／`hashCode`？

順序不能反過來——review 時看得見的是操作，看不見的是 entity 缺了什麼。

### 正確的實作

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    // instanceof rather than getClass(): a Hibernate lazy proxy is a subclass
    if (!(o instanceof Account)) return false;
    // getUuid() rather than the field: a proxy's field is null until initialized
    return getUuid() != null && getUuid().equals(((Account) o).getUuid());
}

@Override
public int hashCode() {
    // 固定值：uuid 於 persist 時才產生，hashCode 若依賴它，
    // 物件加入 HashSet 後會因 hash 改變而在集合中「消失」
    return Account.class.hashCode();
}
```

`uuid` 為 null（尚未持久化）時退回 identity，避免兩個全新物件被視為同一筆。

補之前要先確認兩件事：

1. **該 entity 不會進入大型 `HashSet`／`HashMap`**——固定 `hashCode` 會讓雜湊查找退化為線性掃描。
2. **接受 lazy proxy 被強制初始化**——定義 `equals`／`hashCode` 會讓 Hibernate 的 `overridesEquals` 轉為 true，取消 `BasicLazyInitializer` 對未初始化 proxy 的短路；`@Id` 標在欄位上時 proxy 也無法繞過初始化取 id。對未初始化的 LAZY proxy 做判等，session 內多一次 SELECT、session 外拋 `LazyInitializationException`。改動前雖然不碰 DB，回傳的卻是 identity 比較的錯誤答案——要以 uuid 判等就必須取得對方的 id，這個代價無法迴避。

### 測試要守住的兩件事

判等的測試很容易寫成「永遠不會失敗」：

- **lazy proxy 替身的欄位必須留 null、只有 getter 回傳 id。** 若替身直接把 uuid 寫進真實欄位，判等改成讀欄位（或改用 Lombok `@EqualsAndHashCode(of = "uuid")`）時測試照樣全綠，線上卻退回 identity。
- **要有「物件在 `HashSet` 中時才被指派 uuid」的測試**（模擬 persist），斷言之後仍能 `contains`／`remove`。這是固定 `hashCode` 的唯一理由，缺了它，改成 `Objects.hash(uuid)` 不會被任何斷言擋下。

### 本專案現況（2026-07-28）

82 個 `@Entity` 中已補齊七個：`Account`、`Territory`、`Department`、`School`、`Major104`、`JobCategory104`、`Requisition`。判準是**是否進入依賴判等的集合操作**，而非 entity 的重要性：

- 前六個涵蓋系統中全部十個 `@ManyToMany` 集合欄位，也就是所有「元素對應到有主鍵 join table」的位置。注意其中多數是 `List`（bag）而非 `Set`——bag 不會去重，判等決定的是 `contains`／`remove` 是否命中，以及 bag 重寫時同一筆會不會被寫成兩列；
- `Requisition` 沒有 join table 問題，被 `stream().distinct()` 用在匯入路徑。那些 distinct 目前即使沒有判等也正確（元素來自同一次查詢的同一批實例），補判等是移除「剛好是同一個物件」這個隱含前提。

其餘 entity 只作為 `@ManyToOne` 引用或 owned 的 `@OneToMany` 子實體，不進入依賴判等的集合操作。**新增 `@ManyToMany` 欄位、或對 entity 集合使用 `distinct`／`contains`／`remove` 時，元素 entity 必須一併定義判等。**

> 實際事故：104 履歷匯入因交易拆分而跨 context，`Candidate.hiringManagers` 對 join table 送出重複 INSERT 撞主鍵，整筆履歷 rollback 且每次排程重爆。

---

## 附錄：lazy proxy 的內部機制

上一節的每個設計決定（用 `instanceof`、走 getter、接受 proxy 初始化）都源自這裡。以下以 `ExaminationRequirement.examiner`（`@ManyToOne(fetch = FetchType.LAZY) private Account examiner;`）為例，程式碼證據取自 `hibernate-core-5.6.4.Final`。

### proxy 是什麼

LAZY 關聯不會在載入 owner 時一併查出來，Hibernate 先塞一個**執行期產生的子類別**進去（5.3 之後由 ByteBuddy 生成）：

```java
// 概念示意，實際由 ByteBuddy 產生，無原始碼
public class Account$HibernateProxy$aB3xK extends Account implements HibernateProxy {
    private ByteBuddyInterceptor $$_hibernate_interceptor;   // 唯一有值的欄位

    @Override public String getUuid()  { return interceptor.intercept(this, GET_UUID, NO_ARGS); }
    @Override public boolean equals(Object o) { return interceptor.intercept(this, EQUALS, new Object[]{o}); }
    // …entity 的每一個非 final 方法都被這樣覆寫
}
```

三個關鍵性質：

1. **`instanceof Account` 成立**，但 `getClass()` 回傳的是 `Account$HibernateProxy$aB3xK`——所以判等要用 `instanceof` 而非 `getClass()`。
2. **繼承來的欄位全是 null**（`uuid`、`username`… 都沒有值），只有 interceptor 記著「我是 Account，id 是 X」——所以判等要走 `getUuid()` 而非直接讀欄位。
3. **每個方法都被覆寫成轉交 interceptor**——這是後續所有行為的樞紐。

### `BasicLazyInitializer` 何時被調用

`ByteBuddyInterceptor` 繼承自 `BasicLazyInitializer`。它不是「某個階段跑一次」，而是**每一次呼叫 proxy 上任何方法時**都會進去：

```java
Account examiner = req.getExaminer();   // 只是取欄位，沒有呼叫 proxy 的方法 → 什麼都沒發生
examiner.getName();                     // 呼叫方法 → interceptor 被觸發
```

取得 proxy、判斷 `!= null` 都不碰 DB；要呼叫它的方法才會進攔截器。

### `invoke()` 的分支順序

`BasicLazyInitializer.invoke(Method, Object[], Object proxy)`：

```
方法沒有參數時：
  1. 方法名是 writeReplace                                → 序列化替身
  2. !overridesEquals && 方法名是 hashCode                 → System.identityHashCode(proxy)   ★不碰 DB
  3. isUninitialized() && method == getIdentifierMethod    → getIdentifier()                  ★不碰 DB
  4. 方法名是 getHibernateLazyInitializer                  → this

方法有一個參數時：
  5. !overridesEquals && 方法名是 equals                   → args[0] == proxy                 ★不碰 DB

以上皆不符 → INVOKE_IMPLEMENTATION → getImplementation() → initialize() → SELECT → 轉呼真實物件
```

`overridesEquals` 在建構 proxy 工廠時算好：**這個 entity 類別有沒有自己覆寫 equals**。

`getIdentifierMethod` 是 id 的 getter，**只有 `@Id` 標在 getter 上（property access）時才有值**；標在欄位上時 Hibernate 取不到對應 getter（`GetterFieldImpl.getMethod()` 回 null），第 3 條短路等於關閉。本專案的 `@Id` 都在欄位上。

### 定義 equals 前後的時序對照

```
【定義 equals 前】someSet.add(examinerProxy)
  → proxy.hashCode() → 分支 2 命中 → identityHashCode          不碰 DB
  → proxy.equals(o)  → 分支 5 命中 → (o == proxy)              不碰 DB，但同一筆的兩個物件回傳 false ✗

【定義 equals 後】someSet.add(examinerProxy)
  → proxy.hashCode() → 分支 2、3 皆不命中 → INVOKE_IMPLEMENTATION
                     → initialize() → SELECT * FROM account WHERE uuid = ?
  → proxy.equals(o)  → 同上；且 equals 內部呼叫 ((Account) o).getUuid()，
                       若 o 也是未初始化 proxy，它也被初始化
                                                              查 DB，但答案正確 ✓
```

不是「把對的變慢」，而是「把靜默的錯誤答案換成有代價的正確答案」。

### session 關閉後為何是拋例外

`AbstractLazyInitializer.initialize()` 第一件事就是檢查 session：

```java
if (session == null) {
    throw new LazyInitializationException("could not initialize proxy [" + entityName + "#" + id + "] - no Session");
}
```

proxy 只帶著「我屬於哪個 session」的參照，交易結束後它想查也查不了。完整風險鏈需四個條件同時成立：

```
LAZY @ManyToOne 欄位 → 從未被呼叫過任何 getter（仍未初始化）
  → 被帶出交易邊界（session 關閉）
  → 對它呼叫 equals / hashCode / contains / remove / 放進 HashSet
  → LazyInitializationException
```

### 若要根治：讓 id 走 property access

分支 3 的條件裡**沒有** `overridesEquals`。只要 `getIdentifierMethod` 不是 null，即使覆寫了 equals，proxy 上的 `getUuid()` 仍會走短路直接回傳 id 而不初始化——而 uuid-based 的 equals 做的正是兩側各呼叫一次 `getUuid()`，於是整個判等完全不碰 DB。

作法是 class 維持 `@Access(AccessType.FIELD)`，只在 id 的 getter 上標 `@Access(AccessType.PROPERTY)`。這不是繞路，是啟用 Hibernate 本來就準備好的機制；但需先驗證 5.6 對混合存取與 Lombok 產生的 getter 的實際處理，屬另案評估。

---

## 適用場景

此 pattern 適用於所有需要在主事務中獨立 commit 副作用的場景：

- **AI 別名自動學習** — 匹配成功時自動新增學校別名
- **審計日誌** — 無論主操作成敗，日誌都需要保存
- **通知發送記錄** — 即使主事務失敗，已發送的通知需要記錄
- **計數器更新** — 瀏覽次數等統計不應受主事務影響

## 相關
- [JPA 多表聯集的笛卡兒積](jpa-join-cartesian-product-first-vs-second-level.md) — JPA/交易延伸
