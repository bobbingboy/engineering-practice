---
created: 2026-05-08
tags: [deploy, claude-code, skill, safety-gate, bash]
related:
---

# Deploy Skill Gate Mechanism

記錄 `~/bin/deploy-system` 部署系統如何在 Claude Code 環境下強制走 `/deploy` skill，避免 agent 直接呼叫底層腳本而跳過 skill 內定義的前置安全步驟（git stash、branch check、ahead-of-remote check、SSH 連線測試）。

## 背景與動機

Skill 不只是「方便的入口」，而是承載了**底層腳本不會做的安全步驟**：

| 安全步驟 | 由誰負責 |
|---------|---------|
| Git stash 未提交變更（含 untracked） | Skill (Step 2) |
| Branch 檢查（警告非 main 分支） | Skill (Step 2.1) |
| 本地領先 remote 檢查（避免 SIT 上跑 unpushed commit） | Skill (Step 2.2) |
| SSH 連線測試 | Skill (Step 3) |
| External deps 自動建置（如 下游客製專案-server → 主專案/core） | Script |
| Build / Backup / Upload / Restart | Script |

如果 agent 直接呼叫 `~/bin/deploy` 而跳過 skill，前四項安全檢查就會被跳過——即使 workspace 當下看起來乾淨，分支可能不對、本地可能領先 origin、SSH 可能連不上。

## 閘道位置（重要：只有 deploy.sh 有閘）

實際的閘道函式只實作於：

```
~/bin/deploy-system/scripts/deploy.sh
```

`scripts/lib/build.sh` 是 **被 source 的函式庫**，不是獨立 entry point。所有 build 流程都是 `deploy.sh` 進入點通過閘道後才呼叫 `build_modules()`，因此「閘道只有一處，但保護到了整條 build + deploy 路徑」。

> 想單獨建置不部署？skill 會 source `lib/build.sh` 並呼叫 `build_modules`，這條路徑同樣由 skill 控制，不經過 `deploy.sh` 入口閘，但因為入口被 skill 包住，agent 仍須走 Skill tool。

## 兩道閘的設計

### 閘 1：`check_skill_invocation()`（L266-294）

```bash
if [[ "$CLAUDECODE" != "1" ]]; then
    return 0  # 終端機/CI 直接呼叫不擋
fi
if [[ "$DEPLOY_VIA_SKILL" == "1" ]]; then
    return 0  # 走 skill 進來的，放行
fi

log_gate_event "BYPASS_BLOCKED"
log_error "Direct invocation under Claude Code is not allowed."
# ...提示訊息...
exit 3
```

**判斷邏輯**：

| `CLAUDECODE` | `DEPLOY_VIA_SKILL` | 結果 |
|--------------|-------------------|------|
| 未設定（終端機/CI） | 任意 | 放行 |
| `1` | `1` | 放行 |
| `1` | 未設定/其他 | **exit 3，記錄 `BYPASS_BLOCKED`** |

**錯誤訊息設計**：訊息會指明「請改用 Skill tool with name=deploy」，並明確警告「**不可自行設定 `DEPLOY_VIA_SKILL=1` 繞過閘道**——那會破壞整個安全設計」。這是預防 agent 看到變數名就試著自己設值。

### 閘 2：`check_dirty_workspaces()`（L356-418）

防禦縱深（defense-in-depth）：即使 skill 流程跑過了，仍會再次檢查 workspace 是否乾淨。涵蓋目標專案 workspace **以及所有 `external_deps` 來源 workspace**（例如客製版 server 部署時，會同時檢查客製專案與主專案兩個 workspace）。

```bash
# 收集所有相關 workspace
collect_relevant_workspaces() {
    # 目標 project workspace + 所有 external_deps 來源 workspace（去重）
}

# 檢查每個是否髒
for ws in $(collect_relevant_workspaces); do
    if [[ -n "$(git -C "$ws" status --porcelain)" ]]; then
        dirty+=("$ws")
    fi
done
```

**處理分流**：

| 模式 | 髒 workspace 處理 |
|------|------------------|
| `--dry-run` | 警告但繼續 |
| `--allow-dirty` | 警告並跳過檢查（terminal-only escape hatch） |
| 正式 run | **exit 2，記錄 `DIRTY_BLOCKED`** |

**訊息分流**（依 `CLAUDECODE`）：

- Agent 看到的：「回去用 Skill tool，skill 會自動 stash。**不要手動 stash 也不要用 `--allow-dirty`**」
- 人類看到的：「commit 或 stash 後重跑，或加 `--allow-dirty`」

`--allow-dirty` 在 help 裡就標註「terminal-only escape hatch; do not use in automated/agent runs」。

### Build 殘留自動清理

順帶記一筆：`cleanup_build_residue()`（L326-350）會在 build 階段結束後丟棄 `package-lock.json` 的 build-induced 變更。原因是 frontend maven build 跑 `npm install` 會以「deterministic 但每次略有差異」的方式重寫 lockfile，多專案連續部署時會讓下一輪 dirty-check 誤判。**它不是閘道的一部分，但跟 dirty-check 共生**——沒有它的話，dirty-check 會在第二個專案啟動時被自己 build 出來的東西擋下。

## Skill 端的配合（DEPLOY_VIA_SKILL）

`~/.claude/skills/deploy/SKILL.md` 明確規定：

```bash
# 正確：skill 內所有呼叫 ~/bin/deploy 的 Bash 指令前都必須加上
DEPLOY_VIA_SKILL=1 ~/bin/deploy 主專案 portal sit --dry-run --no-confirm

# 錯誤（會被 exit 3）
~/bin/deploy 主專案 portal sit --dry-run --no-confirm
```

語意上：`DEPLOY_VIA_SKILL=1` 是「**skill 已經執行完前置安全檢查的證明憑證**」。skill 必須先跑完 Step 2（stash）、Step 2.1（branch）、Step 2.2（ahead）、Step 3（SSH），才能把這個憑證帶下去。

**反模式**：在跳過上述步驟的情況下設定 `DEPLOY_VIA_SKILL=1` 來繞過閘道。

## DEPLOY_VIA_SKILL 變數的生命週期

`DEPLOY_VIA_SKILL=1 ~/bin/deploy ...` 用的是 bash **inline environment variable** 語法（command-prefix assignment）。完整機制——fork/exec 模型、與 `export` 的差異、Claude Code Bash tool 為何不持久 shell——這裡只摘要對 deploy gate 而言的關鍵屬性。

**核心性質**：憑證的存活時間 = **一次部署呼叫的 process 生命**

- 呼叫端 shell 從未被設過該變數，**不殘留、不洩漏**
- 下一個 Claude Code Bash tool call 是新 shell，本來就讀不到
- 沒有狀態、沒有 TTL，因為根本沒留下任何狀態可清

**對 skill 流程的意義**：每次呼叫 `~/bin/deploy` 都必須**重新加前綴**，不能「只設一次延用」：

| Skill 步驟 | 變數狀態 |
|-----------|---------|
| Step 4 dry-run `DEPLOY_VIA_SKILL=1 ~/bin/deploy ... --dry-run` | 憑證進入 dry-run 子行程 |
| dry-run 結束 | 憑證隨子行程消失 |
| Step 6 正式部署 | **必須重新加前綴**，否則閘 1 擋下 |

每次部署呼叫都是獨立的「持有憑證 → 用完丟掉」週期，比「skill 啟動時設一次、結束時清掉」更安全——不存在「設了但忘了清」的時間窗。

> **⚠️ 易混淆**：「進入 build 時開啟的 shell 被初始化」這個直覺是錯的。變數不是綁在 shell 啟動上，而是綁在 process invocation 上。

## 事件日誌：skill-gate.log

位置：`~/bin/deploy-system/logs/skill-gate.log`（append-only JSONL）

**Schema**（每行一個 JSON）：

```json
{
  "ts": "2026-05-08T10:48:11+0800",
  "event": "DEPLOY_STARTED",
  "project": "主專案",
  "modules": "server,web,portal",
  "target": "sit",
  "cwd": "/Users/mingchungko/...",
  "claudecode": "1",
  "via_skill": "1",
  "dry_run": "true"
}
```

**事件類型**：

| event | 觸發時機 | 額外欄位 |
|-------|---------|---------|
| `BYPASS_BLOCKED` | 閘 1 擋下時 | （無） |
| `DIRTY_BLOCKED` | 閘 2 擋下時 | `dirty_count` |
| `DEPLOY_STARTED` | 兩道閘都通過、進入部署主流程時 | （無） |
| `DEPLOY_FINISHED` | 主流程結束（trap EXIT，無論成功失敗） | `exit_status` |

**用途**：事後分析「agent 是否曾經繞道？被擋下後有沒有正確改走 skill？」這類安全事件模式。

## 邊界條件與 escape hatch

| 情境 | 行為 |
|------|------|
| 終端機直接跑 `~/bin/deploy` | 兩道閘都不觸發（`CLAUDECODE` 未設） |
| CI / 自動化跑（無 `CLAUDECODE`） | 同上，正常執行 |
| Claude Code agent 直呼 deploy.sh | 閘 1 擋下，exit 3 |
| Skill 呼叫但忘記設 `DEPLOY_VIA_SKILL=1` | 閘 1 擋下（提示應該回去設變數） |
| Skill 沒做 stash 就帶 `DEPLOY_VIA_SKILL=1` 進來 | 閘 1 放行，閘 2 擋下（防禦縱深） |
| 終端機要強制部署髒 workspace | `--allow-dirty` |
| Agent 想用 `--allow-dirty` 繞過 | 不應該——錯誤訊息已明確指引回 skill |

## 為什麼這樣設計？

1. **不影響人類 workflow**：`CLAUDECODE` 只在 Claude Code 環境設定，終端機與 CI 完全不受影響。
2. **不影響 skill workflow**：skill 內統一加前綴即可，是顯式契約。
3. **預防 agent 繞道**：環境變數命名 + 錯誤訊息明確要求「不要自己設值」，加上閘 2 做防禦縱深。
4. **可觀測性**：所有閘道事件落 JSONL log，可重組事件序列。
5. **明確職責分離**：底層腳本不負責「git 安全」，skill 負責；底層腳本只負責「build / backup / upload / restart」。

## 相關


## 相關檔案

- `~/bin/deploy-system/scripts/deploy.sh` — 閘道實作
- `~/bin/deploy-system/scripts/lib/build.sh` — build 函式庫（被 source）
- `~/.claude/skills/deploy/SKILL.md` — skill 定義與 `DEPLOY_VIA_SKILL` 約定
- `~/bin/deploy-system/config/projects.json` — 專案 / 模組 / target 配置
- `~/bin/deploy-system/logs/skill-gate.log` — 閘道事件 JSONL log

## 演進想法（未實作，僅紀錄）

- 若未來想對「直接 source `lib/build.sh` 跑 `build_modules`」也加閘，可在 `build_modules` 開頭做相同的 `CLAUDECODE` + `DEPLOY_VIA_SKILL` 檢查；但目前 skill 內定義的「只 build 不部署」流程就是這條路徑，加閘前需要先想好 skill 端怎麼配合。
- `BYPASS_BLOCKED` 與 `DIRTY_BLOCKED` 後是否真的有人/agent 重新走 skill，可以靠 `cwd` + `project` 把後續的 `DEPLOY_STARTED` 串起來分析。
