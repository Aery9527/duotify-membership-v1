# Tasks: 會員註冊流程

**Feature Branch**: `001-member-registration`  
**Date**: 2025-10-19  
**Status**: Ready for Implementation

本文件提供可執行的任務清單，依照使用者故事優先順序與依賴關係組織，支援 Test-First 開發與增量交付。

---

## 總覽

**總任務數**: 87 個任務  
**User Stories**: 3 個（US1: 基本註冊流程, US2: E-Mail 驗證流程, US3: 未驗證帳號登入）  
**MVP 範圍**: User Story 1（基本註冊流程）  
**預估總工時**: 約 40-50 小時

### 執行策略

- **Test-First 開發**: 每個功能先寫測試再實作
- **增量交付**: 每個 User Story 可獨立測試與部署
- **並行開發**: 標記 [P] 的任務可並行執行
- **MVP 優先**: 建議先完成 User Story 1 作為最小可行產品

---

## 依賴關係圖

```
Phase 1: Setup (必須完成)
    ↓
Phase 2: Foundational (必須完成)
    ↓
Phase 3: User Story 1 - 基本註冊流程 (P1) ← MVP
    ↓
Phase 4: User Story 2 - E-Mail 驗證流程 (P2)
    ↓
Phase 5: User Story 3 - 未驗證帳號登入 (P3)
    ↓
Phase 6: Polish & Cross-Cutting Concerns
```

**Story 相依性**:
- US1 是 US2 和 US3 的前置條件（需要會員註冊才能驗證與登入）
- US2 與 US3 可並行開發（功能獨立）

---

## Phase 1: Setup（專案初始化）

**目標**: 建立專案基礎結構、設定開發環境、安裝依賴套件

**完成標準**: 專案能夠編譯執行，資料庫連線成功，開發工具正常運作

### 專案結構與設定

- [ ] T001 根據 plan.md 建立 Go 專案目錄結構（cmd/, internal/, tests/, config/, migrations/）
- [ ] T002 初始化 Go module 與 go.mod（go mod init duotify-membership-v1）
- [ ] T003 安裝核心依賴套件：Gin, pgx v5, bcrypt, validator v10, testify
- [ ] T004 建立 .env.example 環境變數範本檔案於專案根目錄
- [ ] T005 建立 .gitignore 檔案（排除 .env, bin/, coverage.out 等）
- [ ] T006 建立 Makefile 定義常用指令（build, test, lint, migrate-up, migrate-down）

### 設定檔與配置

- [ ] T007 實作 config 套件載入環境變數（internal/pkg/config/config.go）
- [ ] T008 定義 Config 結構體包含所有設定項目（DB, SMTP, Server, Bcrypt, Verification）
- [ ] T009 實作設定驗證函式確保必要環境變數已設定

### 資料庫設定

- [ ] T010 建立資料庫遷移腳本 001_create_members_and_verifications.sql 於 migrations/ 目錄
- [ ] T011 建立資料庫連線池初始化函式（internal/pkg/database/postgres.go）
- [ ] T012 實作資料庫 health check 函式

### 開發工具設定

- [ ] T013 建立 .golangci.yml golangci-lint 設定檔
- [ ] T014 建立 .air.toml Air hot reload 設定檔
- [ ] T015 建立 docker-compose.yml 定義 PostgreSQL 與 Mailhog 服務

**並行機會**: T007-T009（設定套件）, T010-T012（資料庫）, T013-T015（開發工具）可同時進行

---

## Phase 2: Foundational（基礎建設）

**目標**: 建立所有 User Stories 共用的基礎設施與工具

**完成標準**: 共用元件可獨立測試通過，API 伺服器可啟動

### 核心工具與驗證器

- [ ] T016 實作台灣身分證字號驗證器（internal/pkg/validator/taiwan_id.go）
- [ ] T017 [P] 撰寫身分證驗證器單元測試（tests/unit/validator/taiwan_id_test.go）
- [ ] T018 實作姓名格式驗證器（僅中英文字母，internal/pkg/validator/name.go）
- [ ] T019 [P] 撰寫姓名驗證器單元測試（tests/unit/validator/name_test.go）
- [ ] T020 實作密碼強度驗證器（8-20碼含大小寫數字，internal/pkg/validator/password.go）
- [ ] T021 [P] 撰寫密碼驗證器單元測試（tests/unit/validator/password_test.go）
- [ ] T022 註冊自訂驗證器至 go-playground/validator（taiwanid, chinesename, strongpassword）

### 密碼加密

- [ ] T023 實作密碼加密套件使用 bcrypt（internal/pkg/crypto/password.go）
- [ ] T024 [P] 撰寫密碼加密套件單元測試（tests/unit/crypto/password_test.go）

### 工具函式

- [ ] T025 實作驗證碼生成器（6位數隨機數字，internal/pkg/utils/code_generator.go）
- [ ] T026 [P] 撰寫驗證碼生成器單元測試（tests/unit/utils/code_generator_test.go）
- [ ] T027 實作 UUID 工具函式（internal/pkg/utils/uuid.go）

### HTTP 基礎設施

- [ ] T028 實作標準 JSON 回應結構（internal/api/response/response.go）
- [ ] T029 實作全域錯誤處理中介軟體（internal/api/middleware/error_handler.go）
- [ ] T030 實作結構化日誌中介軟體（internal/api/middleware/logger.go）
- [ ] T031 實作 Recovery 中介軟體（internal/api/middleware/recovery.go）
- [ ] T032 實作請求 ID 產生中介軟體（internal/api/middleware/request_id.go）
- [ ] T033 實作 Router 初始化與中介軟體鏈設定（internal/api/router/router.go）

### E-Mail 服務基礎

- [ ] T034 實作 E-Mail 服務介面定義（internal/service/email.go）
- [ ] T035 實作 SMTP E-Mail 服務實作（internal/service/email_smtp.go）
- [ ] T036 建立 E-Mail 模板（HTML, 驗證碼通知，templates/email/verification_code.html）
- [ ] T037 [P] 撰寫 E-Mail 服務合約測試（tests/contract/email_service_test.go）

### 應用程式進入點

- [ ] T038 實作 main.go 初始化流程（cmd/server/main.go）
- [ ] T039 實作 graceful shutdown 處理

**並行機會**: 
- T016-T021（驗證器）可完全並行
- T023-T024（加密）與 T025-T027（工具）可並行
- T028-T033（HTTP 基礎）可在驗證器完成後並行
- T034-T037（E-Mail）可獨立並行

**測試執行**: `make test-unit` 應全部通過（預期覆蓋率 >80%）

---

## Phase 3: User Story 1 - 基本註冊流程 (P1) ⭐ MVP

**Story 目標**: 使用者可透過填寫身分證字號、姓名、E-Mail、密碼完成註冊，系統建立會員並寄送驗證碼

**獨立測試標準**: 
- ✅ 可透過 POST /api/v1/members/register 成功註冊新會員
- ✅ 身分證字號唯一性驗證（重複註冊被拒絕）
- ✅ 輸入驗證全面（格式檢查、長度限制）
- ✅ 併發註冊安全（同時使用相同身分證號只有一個成功）
- ✅ 驗證碼 E-Mail 成功寄送
- ✅ 密碼安全儲存（bcrypt 加密）

### Domain Layer - Member 實體

- [ ] T040 [US1] 定義 Member 實體結構（internal/domain/member/member.go）
- [ ] T041 [US1] 實作 Member 業務方法：SetPassword, CheckPassword, Lock, Unlock
- [ ] T042 [P] [US1] 撰寫 Member 實體單元測試（tests/unit/domain/member_test.go）

### Domain Layer - VerificationCode 實體

- [ ] T043 [US1] 定義 VerificationCode 實體結構（internal/domain/verification/verification_code.go）
- [ ] T044 [US1] 實作 VerificationCode 業務方法：IsExpired, IsValid, MarkAsUsed, Invalidate
- [ ] T045 [P] [US1] 撰寫 VerificationCode 實體單元測試（tests/unit/domain/verification_code_test.go）

### Repository Layer

- [ ] T046 [US1] 定義 MemberRepository 介面（internal/repository/member.go）
- [ ] T047 [US1] 實作 MemberRepository：Create, GetByNationalID, ExistsByNationalID
- [ ] T048 [US1] 定義 VerificationCodeRepository 介面（internal/repository/verification.go）
- [ ] T049 [US1] 實作 VerificationCodeRepository：Create, InvalidateByMemberID
- [ ] T050 [P] [US1] 撰寫 MemberRepository 單元測試使用 mock（tests/unit/repository/member_test.go）
- [ ] T051 [P] [US1] 撰寫 VerificationCodeRepository 單元測試使用 mock（tests/unit/repository/verification_test.go）

### Service Layer - Registration

- [ ] T052 [US1] 定義 RegistrationService 介面（internal/service/registration.go）
- [ ] T053 [US1] 實作 Register 方法：驗證輸入、檢查重複、建立會員、產生驗證碼、寄送 E-Mail
- [ ] T054 [US1] 實作併發安全處理（捕獲 UNIQUE constraint 違反錯誤）
- [ ] T055 [P] [US1] 撰寫 RegistrationService 單元測試（tests/unit/service/registration_test.go）

### Handler Layer

- [ ] T056 [US1] 定義註冊請求與回應結構（internal/api/handler/registration.go）
- [ ] T057 [US1] 實作 RegisterHandler 處理 POST /api/v1/members/register
- [ ] T058 [US1] 實作輸入驗證與錯誤處理
- [ ] T059 [P] [US1] 撰寫 RegisterHandler 單元測試使用 mock（tests/unit/handler/registration_test.go）

### Router Integration

- [ ] T060 [US1] 註冊 POST /api/v1/members/register 路由（internal/api/router/router.go）

### Integration Tests

- [ ] T061 [US1] 撰寫註冊成功整合測試（tests/integration/registration_success_test.go）
- [ ] T062 [US1] 撰寫重複身分證註冊整合測試（tests/integration/registration_duplicate_test.go）
- [ ] T063 [US1] 撰寫輸入驗證失敗整合測試（tests/integration/registration_validation_test.go）
- [ ] T064 [US1] 撰寫併發註冊整合測試（tests/integration/registration_concurrent_test.go）

**並行機會**:
- T040-T042（Member）與 T043-T045（VerificationCode）可並行
- T046-T047（MemberRepo）與 T048-T049（VerificationRepo）可並行
- T050-T051（Repository 測試）可並行
- T061-T064（整合測試）可在實作完成後並行執行

**完成驗證**:
```bash
# 執行所有測試
make test-unit
make test-integration

# 手動測試
curl -X POST http://localhost:8080/api/v1/members/register \
  -H "Content-Type: application/json" \
  -d '{"national_id":"A123456789","name":"王小明","email":"test@example.com","password":"SecurePass123"}'

# 預期：201 Created, 收到驗證碼 E-Mail
```

**User Story 1 MVP 里程碑**: 完成此階段即可部署最小可行產品 ✅

---

## Phase 4: User Story 2 - E-Mail 驗證流程 (P2)

**Story 目標**: 使用者可使用 6 位數驗證碼完成 E-Mail 驗證，解鎖完整功能，包含驗證碼過期、錯誤次數限制、重發機制

**獨立測試標準**:
- ✅ 可透過 POST /api/v1/members/verify 使用驗證碼啟用帳號
- ✅ 驗證碼過期檢查（5 分鐘）
- ✅ 錯誤嘗試限制（3 次後鎖定 10 分鐘）
- ✅ 驗證碼重發功能（每小時限制 3 次）
- ✅ 舊驗證碼失效機制

### Domain Layer - VerificationResendLog 實體

- [ ] T065 [US2] 定義 VerificationResendLog 實體結構（internal/domain/verification/resend_log.go）
- [ ] T066 [P] [US2] 撰寫 VerificationResendLog 實體單元測試（tests/unit/domain/resend_log_test.go）

### Repository Layer 擴充

- [ ] T067 [US2] 擴充 VerificationCodeRepository：GetLatestByMemberID, IncrementAttempt, MarkAsUsed
- [ ] T068 [US2] 擴充 MemberRepository：UpdateVerificationStatus, LockAccount, UnlockAccount
- [ ] T069 [US2] 定義 VerificationResendLogRepository 介面（internal/repository/resend_log.go）
- [ ] T070 [US2] 實作 VerificationResendLogRepository：Create, CountRecentByMemberID
- [ ] T071 [P] [US2] 撰寫擴充 Repository 方法單元測試（tests/unit/repository/verification_extended_test.go）

### Service Layer - Verification

- [ ] T072 [US2] 定義 VerificationService 介面（internal/service/verification.go）
- [ ] T073 [US2] 實作 VerifyCode 方法：驗證碼檢查、過期檢查、錯誤計數、帳號鎖定
- [ ] T074 [US2] 實作 ResendCode 方法：速率限制檢查、舊碼失效、新碼產生、E-Mail 寄送
- [ ] T075 [P] [US2] 撰寫 VerificationService 單元測試（tests/unit/service/verification_test.go）

### Handler Layer

- [ ] T076 [US2] 實作 VerifyHandler 處理 POST /api/v1/members/verify（internal/api/handler/verification.go）
- [ ] T077 [US2] 實作 ResendHandler 處理 POST /api/v1/members/verify/resend（internal/api/handler/verification.go）
- [ ] T078 [P] [US2] 撰寫 Verification handlers 單元測試（tests/unit/handler/verification_test.go）

### Router Integration

- [ ] T079 [US2] 註冊 POST /api/v1/members/verify 和 /resend 路由（internal/api/router/router.go）

### Integration Tests

- [ ] T080 [US2] 撰寫驗證成功整合測試（tests/integration/verification_success_test.go）
- [ ] T081 [US2] 撰寫驗證碼過期整合測試（tests/integration/verification_expired_test.go）
- [ ] T082 [US2] 撰寫錯誤次數鎖定整合測試（tests/integration/verification_locked_test.go）
- [ ] T083 [US2] 撰寫驗證碼重發整合測試（tests/integration/verification_resend_test.go）
- [ ] T084 [US2] 撰寫重發速率限制整合測試（tests/integration/verification_rate_limit_test.go）

**並行機會**:
- T067-T068（Repository 擴充）與 T069-T070（ResendLog Repo）可並行
- T073-T074（Service 方法）可在 Repository 完成後並行開發
- T080-T084（整合測試）可並行執行

**完成驗證**:
```bash
# 驗證碼驗證
curl -X POST http://localhost:8080/api/v1/members/verify \
  -d '{"member_id":"<uuid>","code":"123456"}'

# 重發驗證碼
curl -X POST http://localhost:8080/api/v1/members/verify/resend \
  -d '{"member_id":"<uuid>"}'

# 預期：驗證成功，帳號狀態更新，舊碼失效
```

---

## Phase 5: User Story 3 - 未驗證帳號登入 (P3)

**Story 目標**: 未驗證使用者可登入但功能受限，驗證後自動解鎖，無需重新登入

**獨立測試標準**:
- ✅ 未驗證使用者可使用帳號密碼登入
- ✅ 登入後顯示驗證狀態提示
- ✅ 進階功能檢查驗證狀態（middleware）
- ✅ 基本瀏覽功能不受限制
- ✅ 驗證成功後立即解鎖

**註**: 完整的登入功能（JWT token 生成、session 管理）可能在其他 feature 實作，此處僅實作驗證狀態檢查機制

### Service Layer - Authentication (簡化版)

- [ ] T085 [US3] 實作 AuthService.Login 方法驗證帳號密碼（internal/service/auth.go）
- [ ] T086 [US3] 實作驗證狀態檢查函式 CheckVerificationStatus
- [ ] T087 [P] [US3] 撰寫 AuthService 單元測試（tests/unit/service/auth_test.go）

### Middleware - Verification Check

- [ ] T088 [US3] 實作驗證狀態檢查中介軟體（internal/api/middleware/verification_check.go）
- [ ] T089 [US3] 實作 RequireVerification 裝飾器供路由使用
- [ ] T090 [P] [US3] 撰寫驗證中介軟體單元測試（tests/unit/middleware/verification_check_test.go）

### Integration Tests

- [ ] T091 [US3] 撰寫未驗證使用者登入整合測試（tests/integration/login_unverified_test.go）
- [ ] T092 [US3] 撰寫進階功能存取限制整合測試（tests/integration/feature_access_test.go）
- [ ] T093 [US3] 撰寫驗證後解鎖整合測試（tests/integration/verification_unlock_test.go）

**並行機會**: T085-T087 與 T088-T090 可並行開發

**完成驗證**:
```bash
# 未驗證使用者登入（簡化測試）
curl -X POST http://localhost:8080/api/v1/auth/login \
  -d '{"email":"test@example.com","password":"SecurePass123"}'

# 預期：登入成功，返回驗證狀態提示
# 嘗試存取進階功能時收到 403 Forbidden
```

**註**: 如果此 feature 不包含完整登入功能，T085-T093 可標記為 OPTIONAL 或延後至登入 feature 實作

---

## Phase 6: Polish & Cross-Cutting Concerns（品質提升）

**目標**: 完善程式碼品質、文件、部署配置、效能最佳化

### 效能最佳化

- [ ] T094 [P] 實作資料庫查詢效能測試（benchmarks）
- [ ] T095 [P] 實作 API 端點效能測試（確保 <200ms p95）
- [ ] T096 優化資料庫連線池參數
- [ ] T097 實作資料庫查詢準備語句（prepared statements）

### 日誌與監控

- [ ] T098 [P] 完善結構化日誌（所有關鍵操作記錄）
- [ ] T099 [P] 實作審計日誌（註冊、驗證、鎖定事件）
- [ ] T100 新增敏感資料遮罩（身分證字號、驗證碼）

### 安全性加固

- [ ] T101 [P] 執行 gosec 安全掃描並修正問題
- [ ] T102 [P] 檢查依賴套件漏洞（go list -m all | nancy sleuth）
- [ ] T103 實作 rate limiting（防止暴力破解）
- [ ] T104 實作 CORS 設定

### 文件完善

- [ ] T105 [P] 補充程式碼 godoc 註解
- [ ] T106 [P] 更新 README.md 包含快速開始指南
- [ ] T107 [P] 建立 API 文件（基於 OpenAPI spec）

### 部署準備

- [ ] T108 建立 Dockerfile（多階段建置）
- [ ] T109 建立 .dockerignore
- [ ] T110 建立 Kubernetes manifests（deployment, service, configmap, secret）
- [ ] T111 建立 CI/CD pipeline 設定（.github/workflows/ci.yml）

### 清理任務

- [ ] T112 執行 gofmt 與 goimports 格式化所有檔案
- [ ] T113 執行 golangci-lint 修正所有 linting 問題
- [ ] T114 確認測試覆蓋率達 80%+（go test -cover ./...）
- [ ] T115 清除未使用的依賴套件（go mod tidy）
- [ ] T116 更新 go.mod 與 go.sum

**並行機會**: T094-T116 多數任務可並行執行（不同檔案或工具）

---

## 並行執行範例

### User Story 1 並行執行建議

**Wave 1** (Domain Layer):
```bash
# 可同時由不同開發者執行
T040-T042: Member 實體與測試
T043-T045: VerificationCode 實體與測試
```

**Wave 2** (Repository Layer):
```bash
T046-T047: MemberRepository
T048-T049: VerificationCodeRepository
# 完成後並行執行測試
T050, T051: Repository 測試
```

**Wave 3** (Service & Handler):
```bash
T052-T054: RegistrationService（依賴 Repository）
T056-T058: RegisterHandler（依賴 Service）
# 測試可並行
T055, T059: Service 與 Handler 測試
```

**Wave 4** (Integration):
```bash
T061, T062, T063, T064: 所有整合測試可並行執行
```

### User Story 2 並行執行建議

**Wave 1** (Repository 擴充):
```bash
T067-T068: 擴充現有 Repository
T069-T070: 新增 ResendLog Repository
T071: 測試
```

**Wave 2** (Service Layer):
```bash
T073: VerifyCode 方法
T074: ResendCode 方法
# 完成後
T075: Service 測試
```

**Wave 3** (Handler & Integration):
```bash
T076-T077: Handlers（可並行）
T078: Handler 測試
# 完成後所有整合測試並行
T080, T081, T082, T083, T084: 整合測試
```

---

## 測試策略

### Test-First 開發流程

每個功能遵循以下順序：

1. **撰寫測試**（測試應失敗）
   ```bash
   # 例如：T017 撰寫身分證驗證器測試
   vim tests/unit/validator/taiwan_id_test.go
   go test ./tests/unit/validator -v
   # 預期：FAIL（尚未實作）
   ```

2. **實作功能**（使測試通過）
   ```bash
   # T016 實作身分證驗證器
   vim internal/pkg/validator/taiwan_id.go
   go test ./tests/unit/validator -v
   # 預期：PASS
   ```

3. **重構**（保持測試通過）
   ```bash
   # 最佳化程式碼，每次修改後執行測試
   go test ./tests/unit/validator -v
   ```

### 測試層級

1. **單元測試** (tests/unit/): 測試單一函式或方法
   - 使用 mock 隔離依賴
   - 執行速度快 (<5秒)
   - 覆蓋率目標 >80%

2. **整合測試** (tests/integration/): 測試 API 端點與資料庫
   - 使用測試資料庫
   - 測試完整請求/回應流程
   - 執行時間 <30秒

3. **合約測試** (tests/contract/): 測試外部服務整合
   - 驗證 E-Mail 服務整合
   - 可使用 mock SMTP server

### 測試執行命令

```bash
# 執行所有單元測試
make test-unit
# 或
go test ./tests/unit/... -v -cover

# 執行所有整合測試
make test-integration
# 或
go test ./tests/integration/... -v

# 執行所有測試
make test-all

# 產生覆蓋率報告
make coverage
# 或
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html
```

---

## 驗收檢查清單

### User Story 1 驗收

- [ ] ✅ 可透過 API 成功註冊新會員
- [ ] ✅ 身分證字號重複註冊被正確拒絕（409 Conflict）
- [ ] ✅ 輸入驗證全面（格式、長度、內容規則）
- [ ] ✅ 併發註冊安全（相同身分證號只有一個成功）
- [ ] ✅ 密碼安全儲存（bcrypt 加密，明文不記錄）
- [ ] ✅ 驗證碼 E-Mail 成功寄送
- [ ] ✅ API 回應格式符合 OpenAPI 規格
- [ ] ✅ 錯誤訊息使用繁體中文
- [ ] ✅ 單元測試覆蓋率 >80%
- [ ] ✅ 整合測試全部通過
- [ ] ✅ 效能測試達標（<200ms p95）

### User Story 2 驗收

- [ ] ✅ 可使用正確驗證碼完成驗證
- [ ] ✅ 驗證碼過期檢查正常（5 分鐘）
- [ ] ✅ 錯誤嘗試限制正常（3 次後鎖定）
- [ ] ✅ 帳號鎖定機制正常（10 分鐘自動解鎖）
- [ ] ✅ 驗證碼重發功能正常
- [ ] ✅ 重發速率限制正常（每小時 3 次）
- [ ] ✅ 舊驗證碼正確失效
- [ ] ✅ 驗證成功後帳號狀態正確更新
- [ ] ✅ 整合測試全部通過

### User Story 3 驗收

- [ ] ✅ 未驗證使用者可登入
- [ ] ✅ 登入後顯示驗證狀態提示
- [ ] ✅ 進階功能正確限制
- [ ] ✅ 基本瀏覽功能不受限
- [ ] ✅ 驗證後自動解鎖
- [ ] ✅ 整合測試全部通過

### 品質檢查

- [ ] ✅ 所有測試通過（unit + integration + contract）
- [ ] ✅ 測試覆蓋率 ≥80%
- [ ] ✅ golangci-lint 無錯誤
- [ ] ✅ gosec 無高危安全問題
- [ ] ✅ 程式碼已格式化（gofmt, goimports）
- [ ] ✅ 效能測試達標（<200ms p95）
- [ ] ✅ 所有 godoc 註解完整
- [ ] ✅ README.md 已更新

### 部署就緒

- [ ] ✅ Dockerfile 建置成功
- [ ] ✅ docker-compose 啟動正常
- [ ] ✅ 資料庫遷移腳本已測試
- [ ] ✅ 環境變數文件完整
- [ ] ✅ CI/CD pipeline 設定完成

---

## 預估工時

| Phase | 任務數 | 預估工時 | 備註 |
|-------|--------|----------|------|
| Phase 1: Setup | 15 | 4-6 小時 | 專案初始化 |
| Phase 2: Foundational | 24 | 10-12 小時 | 共用基礎設施 |
| Phase 3: User Story 1 | 25 | 12-15 小時 | MVP 核心功能 |
| Phase 4: User Story 2 | 20 | 8-10 小時 | 驗證流程 |
| Phase 5: User Story 3 | 9 | 3-4 小時 | 功能限制（簡化） |
| Phase 6: Polish | 23 | 6-8 小時 | 品質提升 |
| **總計** | **116** | **43-55 小時** | 約 1-1.5 週（單人） |

**MVP 工時**: Phase 1 + Phase 2 + Phase 3 = 約 26-33 小時（3-4 天）

---

## 建議實作順序

### 第一優先：MVP (User Story 1)

```
Phase 1 (Setup) → Phase 2 (Foundational) → Phase 3 (User Story 1)
```

完成後可部署最小可行產品，使用者可註冊並收到驗證碼。

### 第二優先：完整驗證功能 (User Story 2)

```
Phase 4 (User Story 2)
```

完成後使用者可完成驗證流程，解鎖完整功能。

### 第三優先：使用者體驗提升 (User Story 3)

```
Phase 5 (User Story 3)
```

允許未驗證使用者登入並使用基本功能。

### 最後：品質與部署 (Polish)

```
Phase 6 (Polish)
```

完善文件、最佳化效能、準備生產部署。

---

## 快速開始

### 開始實作 MVP

```bash
# 1. 切換到 feature branch
git checkout 001-member-registration

# 2. 開始 Phase 1 - Setup
# 執行 T001: 建立專案結構
mkdir -p cmd/server internal/{api/{handler,middleware,router,response},domain/{member,verification},repository,service,pkg/{config,crypto,database,utils,validator}} tests/{unit,integration,contract} config migrations templates/email

# 3. 初始化 Go module
go mod init duotify-membership-v1

# 4. 繼續執行後續任務...
# 參考 quickstart.md 進行環境設定

# 5. 測試驅動開發
# 每個功能先寫測試再實作
```

### 並行開發建議

如果有多位開發者，可按以下方式分工：

- **開發者 A**: Phase 2 驗證器 (T016-T022)
- **開發者 B**: Phase 2 HTTP 基礎設施 (T028-T033)
- **開發者 C**: Phase 2 E-Mail 服務 (T034-T037)

完成 Phase 2 後：

- **開發者 A**: Phase 3 Domain Layer (T040-T045)
- **開發者 B**: Phase 3 Repository Layer (T046-T051)
- **開發者 C**: Phase 3 Service Layer (T052-T055)

---

## 相關文件

- [Feature Specification](./spec.md) - 功能需求與驗收標準
- [Implementation Plan](./plan.md) - 技術架構與設計決策
- [Data Model](./data-model.md) - 資料庫 schema 與實體定義
- [API Contracts](./contracts/openapi.yaml) - API 端點規格
- [Research Notes](./research.md) - 技術選型理由
- [Quickstart Guide](./quickstart.md) - 開發環境設定

---

## 支援與協助

遇到問題時：

1. **查閱文件**: 先檢查 quickstart.md 常見問題章節
2. **執行測試**: `make test-all` 確認現有功能正常
3. **查看日誌**: `tail -f logs/app.log | jq .`
4. **團隊協助**: 在專案中開 Issue 或聯繫團隊

祝實作順利！🚀
