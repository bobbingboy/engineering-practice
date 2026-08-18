---
name: commit
description: 使用團隊 commit message 規範提交程式碼變更。當使用者要求 commit、提交、送 commit 時觸發。
argument-hint: "[可選：額外說明或指定 commit type]"
---

# Commit

Create a commit following the team's commit message conventions.

## Instructions

1. Run `git status` and `git diff` to understand what changes exist
2. Stage the relevant changed files (only files related to the current task)
3. Analyze the changes and determine the appropriate commit type based on the conventions below
4. Generate a commit message in Traditional Chinese (繁體中文) following the format
5. Execute the commit
6. After commit, show the commit hash and message, then wait for user to request push

## Commit Message Conventions

### Initial
Project, module, or folder initialization.
- Format: `Initial`

### Feature
Add new functionality.
- Format: `Feature: [#功能代號|#問題單號|功能名稱|功能說明]`
- Examples:
  - `Feature: #UC-IDM-01` (Use Case code)
  - `Feature: #US-IDM-01` (User Story code)
  - `Feature: 帳號管理` (Feature name)
  - `Feature: 帳號的新增、修改、刪除與查詢` (Feature description)

### Enhancement
Improve existing functionality to be more complete, efficient, or secure.
- Format: `Enhancement: [#功能代號|#問題單號|功能名稱|增強說明]`
- Examples:
  - `Enhancement: #UC-IDM-01`
  - `Enhancement: 能夠使用角色來指定使用者權限`

### Change
Modify existing functionality.
- Format: `Change: [#功能代號|#問題單號|功能名稱|變更說明]`

### Bug
Fix program errors/bugs.
- Format: `Bug: [#功能代號|#問題單號|功能名稱|錯誤說明]`
- Examples:
  - `Bug: #BUG-123`
  - `Bug: 修正登入後未正確跳轉問題`

### UX
Adjust UI to improve user experience.
- Format: `UX: [#功能代號|#問題單號|功能名稱|改善說明]`

### Security
Security-related updates.
- Format: `Security: [#功能代號|#問題單號|功能名稱|改善說明]`

### Configuration
System configuration.
- Format: `Configuration: [配置說明]`

### Branding
System branding/identity configuration.
- Format: `Branding: [配置說明]`

### Refactoring
Restructure code without changing functionality.
- Format: `Refactoring: [#功能代號|#問題單號|功能名稱|重構說明]`

### Documentation
Submit documentation.
- Format: `Documentation: [Requirements|Design|Test|Management|Environment|Education]`
- Categories:
  - Requirements: Requirement documents
  - Design: Design documents
  - Test: Test documents
  - Management: Project management documents
  - Environment: Operation/development environment documents
  - Education: User training, manuals, etc.

### Environment
Submit operation or development environment files.
- Format: `Environment: [DEV|SIT|UAT|PROD]`
- Categories:
  - DEV: Development environment
  - SIT: System Integration Testing environment
  - UAT: User Acceptance Testing environment
  - PROD: Production environment

### Release
Release-related files.
- Format: `Release: [版本編號|發佈說明]`
- Example: `Release: v1.2.0`

### Test
Test-related files.
- Format: `Test: [#功能代號|功能名稱|測試說明]`

### Upgrade
Large-scale feature upgrades.
- Format: `Upgrade: [升級說明]`

### Undo
Revert files.
- Format: `Undo: [復原說明]`

## Commit Execution

When creating the commit, always use this format with HEREDOC:
```bash
git commit -m "$(cat <<'EOF'
[Commit Type]: [說明]

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

## Additional Context from User

$ARGUMENTS
