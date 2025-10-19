# Implementation Plan: 會員註冊流程

**Branch**: `001-member-registration` | **Date**: 2025-10-19 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-member-registration/spec.md`

**⚠️ LANGUAGE REQUIREMENT**: This plan MUST be written in Traditional Chinese (zh-TW) per Constitution Principle V.

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

實作會員註冊與 E-Mail 驗證流程，使用者透過填寫身分證字號、姓名、E-Mail、密碼完成註冊，系統寄送 6 位數驗證碼進行 E-Mail 驗證。未驗證使用者可登入但功能受限（限制進階功能如付費內容、個人化設定、社交互動）。技術實作使用 Go 1.25.0 與 Gin 框架建立 RESTful API，包含輸入驗證、併發安全、驗證碼管理、帳號鎖定機制等核心功能。

## Technical Context

**Language/Version**: Go 1.25.0  
**Primary Dependencies**: Gin (HTTP server), pgx v5 (PostgreSQL driver), bcrypt (密碼加密), go-playground/validator v10 (輸入驗證), testify (測試斷言與 mock)  
**Storage**: PostgreSQL 14+ (關聯式資料庫，支援 ACID 交易與 UNIQUE constraint)  
**Testing**: Go 標準 testing 套件 + testify/assert + testify/mock  
**Target Platform**: Linux server (容器化部署)
**Project Type**: Web API (單一後端服務)  
**Performance Goals**: 讀取操作 <100ms p95, 寫入操作 <200ms p95, 支援 1,000+ 並發請求  
**Constraints**: 回應時間 <200ms p95 (註冊), <100ms p95 (驗證), 記憶體使用 <512MB, 驗證碼有效期限 5 分鐘, 錯誤嘗試鎖定 10 分鐘  
**Scale/Scope**: 預期 10,000+ 使用者, 3 個主要 API 端點（註冊、驗證、重發驗證碼）, 支援每小時 1,000+ 註冊請求

**Additional Services**: SMTP (使用 Go net/smtp + 第三方 SMTP 服務如 SendGrid/AWS SES 發送驗證碼 E-Mail)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Verify compliance with constitution principles (`.specify/memory/constitution.md`):

**I. Code Quality & Maintainability**
- [x] Design follows idiomatic Go patterns and Effective Go guidelines
- [x] Error handling strategy defined (all errors explicitly handled)
- [x] Logging approach specified (structured logging with context)
- [x] Functions kept modular (<50 lines guideline)
- [x] Code review plan includes gofmt, goimports, go vet checks

**II. Testing Standards (NON-NEGOTIABLE)**
- [x] Test-first approach confirmed (tests written before implementation)
- [x] Unit test plan achieves 80%+ coverage target
- [x] Integration tests defined for API endpoints and database operations
- [x] Contract tests planned for external service integrations (E-Mail 服務)
- [x] Test execution time within limits (unit <5s, integration <30s)

**III. User Experience Consistency**
- [x] API responses use standard JSON structure (success/error format)
- [x] HTTP status codes semantically correct
- [x] Input validation comprehensive (all fields validated)
- [x] Error messages user-friendly and actionable
- [x] API versioning strategy defined (path-based, e.g., /api/v1/)

**IV. Performance Requirements**
- [x] Response time targets defined (read <100ms p95, write <200ms p95)
- [x] Scalability design supports 1,000+ concurrent requests
- [x] Database queries optimized (indexes planned, no N+1)
- [x] Connection pooling implemented for external services
- [x] Context timeouts configured (max 30s)
- [x] Pagination implemented for list endpoints (max 100 items) - N/A for this feature

**V. Documentation Language Standard (NON-NEGOTIABLE)**
- [x] Specification written in Traditional Chinese (zh-TW)
- [x] Implementation plan written in Traditional Chinese (zh-TW)
- [x] User-facing documentation written in Traditional Chinese (zh-TW)
- [x] Code comments may be in English; code identifiers must be in English

**Quality Gates**
- [x] Pre-commit checks planned (formatting, linting, local tests)
- [x] CI/CD pipeline includes coverage and security checks
- [x] Performance benchmarks defined for critical paths

## Project Structure

### Documentation (this feature)

```
specs/001-member-registration/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── openapi.yaml     # OpenAPI 3.0 API 規格
│   └── examples/        # API 請求/回應範例
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```
# Single project structure (Go web API)
cmd/
└── server/
    └── main.go          # 應用程式進入點

internal/
├── api/
│   ├── handler/         # HTTP 處理器（註冊、驗證、重發）
│   ├── middleware/      # 中介軟體（logging, recovery, validation）
│   └── router/          # 路由設定
├── domain/
│   ├── member/          # 會員實體與業務邏輯
│   └── verification/    # 驗證碼實體與業務邏輯
├── repository/          # 資料存取層
│   ├── member.go
│   └── verification.go
├── service/             # 業務服務層
│   ├── registration.go  # 註冊服務
│   ├── verification.go  # 驗證服務
│   └── email.go         # E-Mail 服務
└── pkg/
    ├── validator/       # 輸入驗證（身分證、姓名、密碼）
    ├── crypto/          # 密碼加密
    └── utils/           # 工具函式

tests/
├── contract/            # 外部服務合約測試（E-Mail 服務）
├── integration/         # API 整合測試
└── unit/                # 單元測試

config/
└── config.yaml          # 應用程式設定檔

migrations/              # 資料庫遷移腳本
└── 001_create_members_and_verifications.sql
```

**Structure Decision**: 選擇單一專案結構（Single project），因為此功能為純後端 API 服務，無前端或多平台需求。採用 Go 標準專案佈局，將應用程式邏輯放在 `internal/` 目錄確保封裝性，使用分層架構（handler → service → repository）實現關注點分離與可測試性。

## Complexity Tracking

*No violations - all Constitution principles satisfied*

---

## Phase 0: Research - 完成 ✅

**產出文件**: [research.md](./research.md)

已完成所有技術選型研究，解決所有 NEEDS CLARIFICATION 項目：

| 項目 | 決策 | 理由 |
|------|------|------|
| 資料庫 | PostgreSQL 14+ | ACID 支援、UNIQUE constraint 效能、成熟的 Go 生態系 |
| 資料庫驅動 | pgx v5 | 原生 PostgreSQL 驅動、效能最佳、連線池管理完善 |
| E-Mail 服務 | net/smtp + SMTP provider | 標準函式庫、靈活切換服務商、易於測試 |
| 密碼加密 | bcrypt (cost=12) | 業界標準、自動加鹽、符合憲法安全要求 |
| 輸入驗證 | go-playground/validator v10 | 宣告式驗證、Gin 整合良好、支援自訂驗證器 |
| 測試框架 | testing + testify | 零依賴核心、testify 增強可讀性、符合 Go 慣例 |

所有選擇符合 Constitution 原則並支援效能目標（<200ms p95）。

---

## Phase 1: Design & Contracts - 完成 ✅

### 資料模型設計

**產出文件**: [data-model.md](./data-model.md)

完成三個核心實體設計：

1. **Member (會員帳戶)**
   - 欄位：id, national_id (UNIQUE), name, email, password_hash, is_verified, is_locked, locked_until, created_at, updated_at
   - 索引：主鍵、唯一索引（national_id）、一般索引（email）、複合索引（鎖定狀態）
   - 業務方法：SetPassword, CheckPassword, Lock, Unlock, IsCurrentlyLocked

2. **VerificationCode (驗證碼)**
   - 欄位：id, member_id (FK), code, status, attempt_count, created_at, expires_at, used_at
   - 狀態：unused, used, expired, invalidated
   - 索引：主鍵、外鍵索引、複合索引（member_id + status + created_at）
   - 業務方法：IsExpired, IsValid, IncrementAttempt, MarkAsUsed, Invalidate

3. **VerificationResendLog (重發記錄)**
   - 欄位：id, member_id (FK), resent_at
   - 用途：實施每小時 3 次的速率限制

**關鍵設計決策**：
- 使用 UUID 作為主鍵（分散式友善）
- UNIQUE constraint 於身分證字號（處理併發註冊）
- 外鍵 CASCADE DELETE（自動清理關聯資料）
- 索引策略最佳化查詢效能
- 資料庫遷移腳本已準備

### API 合約設計

**產出文件**: [contracts/openapi.yaml](./contracts/openapi.yaml)

完成 3 個 RESTful API 端點設計：

1. **POST /api/v1/members/register** - 註冊新會員
   - Request: national_id, name, email, password
   - Response 201: member_id, email, is_verified, message
   - Errors: 400 (驗證錯誤), 409 (身分證已註冊), 500 (伺服器錯誤)

2. **POST /api/v1/members/verify** - 驗證 E-Mail
   - Request: member_id, code
   - Response 200: member_id, is_verified, message
   - Errors: 400 (驗證碼錯誤/過期), 403 (帳號鎖定), 404 (會員不存在), 500

3. **POST /api/v1/members/verify/resend** - 重新寄送驗證碼
   - Request: member_id
   - Response 200: member_id, message
   - Errors: 404 (會員不存在), 429 (超過重發限制), 500

**API 設計原則**：
- 遵循 RESTful 規範
- 統一 JSON 回應格式（success + data/error）
- HTTP 狀態碼語意正確
- 錯誤訊息使用繁體中文（符合憲法）
- 包含詳細的範例與說明

### 快速開始指南

**產出文件**: [quickstart.md](./quickstart.md)

涵蓋完整的開發流程：

- **環境設定**：Go 1.25.0、PostgreSQL 14+、必要工具安裝
- **專案設定**：Clone、依賴安裝、資料庫建立、環境變數設定
- **開發流程**：啟動伺服器、測試 API、使用 Postman/Insomnia
- **測試**：單元測試、整合測試、合約測試、效能測試、Test-First 範例
- **程式碼品質**：Linting、格式化、靜態分析、安全掃描
- **開發工具**：Mailhog（E-Mail 測試）、Air（Hot Reload）、資料庫管理工具
- **除錯**：日誌查看、資料庫查詢、Delve 除錯器
- **常見問題**：5 個常見問題與解決方法
- **部署**：Docker 部署、生產環境檢查清單
- **效能最佳化**：資料庫與應用層最佳化建議

### Agent Context 更新

**產出文件**: `.github/copilot-instructions.md`

已自動更新 GitHub Copilot 上下文檔案，包含：
- 語言：Go 1.25.0
- 框架：Gin, pgx v5, bcrypt, go-playground/validator v10, testify
- 資料庫：PostgreSQL 14+
- 專案類型：Web API

---

## Phase 2: Task Decomposition - 待執行

Phase 2 將由 `/speckit.tasks` 命令執行，產出：
- **tasks.md**: 可執行的任務分解與實作順序
- 每個任務包含：描述、驗收標準、預估工時、依賴關係

執行命令：
```bash
/speckit.tasks
```

---

## Constitution Check - 重新驗證 ✅

Phase 1 設計完成後重新驗證憲法合規性：

**I. Code Quality & Maintainability** ✅
- 分層架構設計（handler → service → repository）遵循 Go 慣例
- 錯誤處理策略明確（所有錯誤明確處理與記錄）
- 結構化日誌（使用 context 傳遞 request_id）
- 函式保持模組化（<50 行指引）

**II. Testing Standards** ✅
- Test-First 開發流程已規劃
- 測試結構完整（unit / integration / contract）
- 覆蓋率目標 80%+
- 測試執行時間符合限制

**III. User Experience Consistency** ✅
- API 回應格式統一（success/error 結構）
- HTTP 狀態碼語意正確
- 輸入驗證全面（所有欄位驗證）
- 錯誤訊息繁體中文且可操作
- API 版本策略明確（/api/v1/）

**IV. Performance Requirements** ✅
- 回應時間目標已定義並可達成
- 支援 1,000+ 並發請求（無狀態設計）
- 資料庫查詢最佳化（索引策略、避免 N+1）
- 連線池已規劃
- Context timeout 已設定（5s API, 2s DB）

**V. Documentation Language Standard** ✅
- 所有規格與計畫文件使用繁體中文
- API 文件與錯誤訊息使用繁體中文
- 程式碼識別使用英文
- 註解可使用英文

**Quality Gates** ✅
- Pre-commit 檢查已規劃（gofmt, golangci-lint, tests）
- CI/CD 流程已定義
- 效能基準測試已規劃

**結論**：所有憲法原則已滿足，無違規項目，可進入 Phase 2 任務分解階段。

