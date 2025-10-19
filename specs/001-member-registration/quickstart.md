# Quickstart Guide - 會員註冊與驗證

**Feature**: 001-member-registration  
**Date**: 2025-10-19

本文件提供開發者快速上手會員註冊與驗證功能的指南，包含環境設定、開發流程、測試方法與部署步驟。

---

## 前置需求

### 開發環境

- **Go 1.25.0+**: [安裝指南](https://go.dev/doc/install)
- **PostgreSQL 14+**: [安裝指南](https://www.postgresql.org/download/)
- **Git**: 版本控制
- **IDE**: 建議使用 GoLand 或 VS Code + Go 擴充套件

### 必要工具

```bash
# 安裝開發工具
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# 驗證安裝
go version          # 應顯示 go1.25.0 或更高
psql --version      # 應顯示 PostgreSQL 14 或更高
golangci-lint --version
```

---

## 專案設定

### 1. Clone 專案

```bash
git clone <repository-url>
cd duotify-membership-v1
git checkout 001-member-registration
```

### 2. 安裝依賴

```bash
go mod download
go mod verify
```

### 3. 資料庫設定

#### 建立資料庫

```bash
# 連線到 PostgreSQL
psql -U postgres

# 建立資料庫與使用者
CREATE DATABASE duotify_membership;
CREATE USER duotify_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE duotify_membership TO duotify_user;
\q
```

#### 執行資料庫遷移

```bash
# 設定環境變數
export DATABASE_URL="postgres://duotify_user:your_secure_password@localhost:5432/duotify_membership?sslmode=disable"

# 執行遷移
migrate -path ./migrations -database $DATABASE_URL up

# 驗證資料表已建立
psql -U duotify_user -d duotify_membership -c "\dt"
# 應顯示: members, verification_codes, verification_resend_logs
```

### 4. 環境變數設定

建立 `.env` 檔案（請勿提交到 Git）：

```bash
# 伺服器設定
SERVER_HOST=localhost
SERVER_PORT=8080
GIN_MODE=debug

# 資料庫設定
DB_HOST=localhost
DB_PORT=5432
DB_NAME=duotify_membership
DB_USER=duotify_user
DB_PASSWORD=your_secure_password
DB_MAX_CONNS=20
DB_MIN_CONNS=2

# SMTP 設定（使用 SendGrid 範例）
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USERNAME=apikey
SMTP_PASSWORD=your_sendgrid_api_key
SMTP_FROM=noreply@duotify.com
SMTP_FROM_NAME=Duotify

# 密碼加密設定
BCRYPT_COST=12

# 驗證碼設定
VERIFICATION_CODE_LENGTH=6
VERIFICATION_CODE_EXPIRY_MINUTES=5
VERIFICATION_MAX_ATTEMPTS=3
VERIFICATION_LOCK_DURATION_MINUTES=10
VERIFICATION_RESEND_LIMIT_PER_HOUR=3

# 日誌設定
LOG_LEVEL=debug
LOG_FORMAT=json
```

---

## 開發流程

### 1. 啟動開發伺服器

```bash
# 方式一：直接執行
go run cmd/server/main.go

# 方式二：建置後執行
go build -o bin/server cmd/server/main.go
./bin/server

# 預期輸出：
# {"level":"info","time":"2025-10-19T10:51:55Z","msg":"Server starting on :8080"}
# {"level":"info","time":"2025-10-19T10:51:55Z","msg":"Database connected"}
```

### 2. 測試 API

#### 註冊新會員

```bash
curl -X POST http://localhost:8080/api/v1/members/register \
  -H "Content-Type: application/json" \
  -d '{
    "national_id": "A123456789",
    "name": "王小明",
    "email": "wang@example.com",
    "password": "SecurePass123"
  }'

# 預期回應：
# {
#   "success": true,
#   "data": {
#     "member_id": "123e4567-e89b-12d3-a456-426614174000",
#     "email": "wang@example.com",
#     "is_verified": false,
#     "message": "註冊成功，請至信箱收取驗證碼"
#   }
# }
```

#### 驗證 E-Mail

```bash
# 從 E-Mail 取得驗證碼（開發環境可查看 SMTP 日誌或使用 Mailhog）
curl -X POST http://localhost:8080/api/v1/members/verify \
  -H "Content-Type: application/json" \
  -d '{
    "member_id": "123e4567-e89b-12d3-a456-426614174000",
    "code": "123456"
  }'

# 預期回應：
# {
#   "success": true,
#   "data": {
#     "member_id": "123e4567-e89b-12d3-a456-426614174000",
#     "is_verified": true,
#     "message": "驗證成功"
#   }
# }
```

#### 重新寄送驗證碼

```bash
curl -X POST http://localhost:8080/api/v1/members/verify/resend \
  -H "Content-Type: application/json" \
  -d '{
    "member_id": "123e4567-e89b-12d3-a456-426614174000"
  }'

# 預期回應：
# {
#   "success": true,
#   "data": {
#     "member_id": "123e4567-e89b-12d3-a456-426614174000",
#     "message": "驗證碼已重新寄送至您的信箱"
#   }
# }
```

### 3. 使用 Postman/Insomnia

匯入 OpenAPI 規格檔案以自動產生 API 集合：

```bash
# OpenAPI 規格檔案位置
specs/001-member-registration/contracts/openapi.yaml
```

**Postman 匯入步驟**：
1. 開啟 Postman
2. 點選 Import → Upload Files
3. 選擇 `openapi.yaml`
4. 自動產生完整的 API 集合與範例

---

## 測試

### 執行單元測試

```bash
# 執行所有單元測試
go test ./tests/unit/... -v -cover

# 執行特定套件測試
go test ./internal/pkg/validator -v -cover

# 產生覆蓋率報告
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out -o coverage.html

# 開啟瀏覽器查看覆蓋率
open coverage.html  # macOS
xdg-open coverage.html  # Linux
start coverage.html  # Windows
```

### 執行整合測試

```bash
# 設定測試資料庫
export TEST_DATABASE_URL="postgres://duotify_user:your_password@localhost:5432/duotify_membership_test?sslmode=disable"

# 建立測試資料庫
createdb -U postgres duotify_membership_test
migrate -path ./migrations -database $TEST_DATABASE_URL up

# 執行整合測試
go test ./tests/integration/... -v

# 清理測試資料庫
dropdb -U postgres duotify_membership_test
```

### 執行合約測試

```bash
# 測試 E-Mail 服務整合
go test ./tests/contract/... -v
```

### 執行效能測試

```bash
# 執行基準測試
go test ./internal/service -bench=. -benchmem -benchtime=10s

# 預期結果應符合憲法要求：
# - 註冊操作 <200ms p95
# - 驗證操作 <100ms p95
```

### Test-First 開發範例

```bash
# 1. 先寫測試（應該失敗）
vim internal/pkg/validator/taiwan_id_test.go

# 2. 執行測試確認失敗
go test ./internal/pkg/validator -v
# 預期：FAIL（因為尚未實作）

# 3. 實作功能
vim internal/pkg/validator/taiwan_id.go

# 4. 執行測試確認通過
go test ./internal/pkg/validator -v
# 預期：PASS

# 5. 重構（保持測試通過）
# 進行程式碼最佳化，每次修改後執行測試確保無迴歸
```

---

## 程式碼品質檢查

### Linting

```bash
# 執行 golangci-lint（包含多種檢查工具）
golangci-lint run ./...

# 自動修正可修正的問題
golangci-lint run --fix ./...
```

### 格式化

```bash
# 格式化程式碼
gofmt -w .
goimports -w .

# 驗證格式化
gofmt -l .  # 應無輸出
```

### Static Analysis

```bash
# 執行 go vet
go vet ./...

# 檢查潛在錯誤
staticcheck ./...
```

### 安全性掃描

```bash
# 安裝 gosec
go install github.com/securego/gosec/v2/cmd/gosec@latest

# 執行安全掃描
gosec ./...
```

---

## 開發工具

### 使用 Mailhog 測試 E-Mail

Mailhog 是本地 SMTP 測試工具，可攔截並顯示所有發送的 E-Mail。

```bash
# 安裝 Mailhog（macOS）
brew install mailhog

# 安裝 Mailhog（Linux）
go install github.com/mailhog/MailHog@latest

# 啟動 Mailhog
mailhog

# Web UI: http://localhost:8025
# SMTP Port: 1025
```

修改 `.env` 使用 Mailhog：

```bash
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USERNAME=
SMTP_PASSWORD=
```

### 使用 Air 實現 Hot Reload

```bash
# 安裝 Air
go install github.com/cosmtrek/air@latest

# 建立 .air.toml 設定檔
air init

# 啟動 Air（程式碼變更時自動重新編譯）
air

# 或使用專案提供的設定
air -c .air.toml
```

### 資料庫管理工具

推薦使用以下工具之一：

- **DBeaver**: 跨平台資料庫工具
- **TablePlus**: macOS/Windows 輕量工具
- **pgAdmin**: PostgreSQL 官方工具
- **psql**: 命令列工具（已隨 PostgreSQL 安裝）

---

## 除錯

### 查看應用程式日誌

```bash
# 即時查看日誌（structured logging）
tail -f logs/app.log | jq .

# 過濾特定層級
tail -f logs/app.log | jq 'select(.level=="error")'

# 追蹤特定 request_id
tail -f logs/app.log | jq 'select(.request_id=="req-123456")'
```

### 查看資料庫狀態

```bash
# 連線資料庫
psql -U duotify_user -d duotify_membership

# 查看會員資料
SELECT id, national_id, name, email, is_verified, is_locked FROM members;

# 查看驗證碼記錄
SELECT member_id, code, status, attempt_count, expires_at FROM verification_codes 
WHERE member_id = 'your-member-id' 
ORDER BY created_at DESC LIMIT 5;

# 查看重發記錄
SELECT member_id, resent_at FROM verification_resend_logs 
WHERE member_id = 'your-member-id' AND resent_at > NOW() - INTERVAL '1 hour';

# 清除測試資料
TRUNCATE TABLE members CASCADE;  -- 同時清除關聯的 verification_codes 和 resend_logs
```

### 使用 Delve 除錯器

```bash
# 安裝 Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# 啟動除錯模式
dlv debug cmd/server/main.go

# 在 Delve 中設定中斷點
(dlv) break internal/service/registration.go:42
(dlv) continue

# 檢查變數
(dlv) print member
(dlv) locals
```

---

## 常見問題

### Q1: 資料庫連線失敗

**錯誤訊息**：`pq: role "duotify_user" does not exist`

**解決方法**：
```bash
# 確認資料庫使用者已建立
psql -U postgres -c "\du"

# 若未建立，執行：
psql -U postgres -c "CREATE USER duotify_user WITH PASSWORD 'your_password';"
psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE duotify_membership TO duotify_user;"
```

### Q2: E-Mail 未發送

**可能原因**：
- SMTP 設定錯誤
- SMTP 服務商需要啟用

**除錯步驟**：
```bash
# 檢查 SMTP 連線
telnet smtp.sendgrid.net 587

# 查看應用程式日誌
tail -f logs/app.log | grep "email"

# 使用 Mailhog 測試本地發送
```

### Q3: 驗證碼一直錯誤

**可能原因**：
- 驗證碼已過期（5 分鐘）
- 使用舊的驗證碼（重發後舊碼失效）

**除錯步驟**：
```sql
-- 查看當前有效的驗證碼
SELECT code, status, expires_at FROM verification_codes 
WHERE member_id = 'your-member-id' AND status = 'unused'
ORDER BY created_at DESC LIMIT 1;

-- 檢查是否過期
SELECT NOW() > expires_at as is_expired FROM verification_codes 
WHERE member_id = 'your-member-id' AND status = 'unused'
ORDER BY created_at DESC LIMIT 1;
```

### Q4: 帳號被鎖定

**錯誤訊息**：`ACCOUNT_LOCKED`

**解決方法**：
```sql
-- 手動解鎖帳號（測試用）
UPDATE members SET is_locked = FALSE, locked_until = NULL 
WHERE id = 'your-member-id';

-- 或等待 10 分鐘自動解鎖
```

### Q5: 身分證字號格式驗證失敗

**常見錯誤**：
- 使用小寫英文字母（應為大寫）
- 檢查碼計算錯誤

**測試用有效身分證字號**：
- A123456789
- B234567890
- C345678901

---

## 部署

### 本地 Docker 部署

```bash
# 建置 Docker image
docker build -t duotify-membership:latest .

# 啟動服務（使用 docker-compose）
docker-compose up -d

# 查看日誌
docker-compose logs -f app

# 停止服務
docker-compose down
```

### 生產環境檢查清單

- [ ] 環境變數已正確設定（使用 Kubernetes Secrets 或類似機制）
- [ ] 資料庫遷移已執行
- [ ] 資料庫連線池參數已調整
- [ ] SMTP 服務已設定並測試
- [ ] 日誌層級設定為 `info` 或 `warn`（非 `debug`）
- [ ] Health check 端點已設定
- [ ] 監控與告警已設定（Prometheus/Grafana）
- [ ] 備份策略已實施
- [ ] SSL/TLS 已啟用
- [ ] Rate limiting 已設定（nginx/API Gateway）
- [ ] CORS 政策已設定

---

## 效能最佳化建議

### 資料庫

```sql
-- 定期執行 VACUUM 清理
VACUUM ANALYZE members;
VACUUM ANALYZE verification_codes;

-- 查看索引使用狀況
SELECT schemaname, tablename, indexname, idx_scan 
FROM pg_stat_user_indexes 
ORDER BY idx_scan;

-- 查看慢查詢
SELECT query, mean_exec_time, calls 
FROM pg_stat_statements 
WHERE mean_exec_time > 100  -- 超過 100ms
ORDER BY mean_exec_time DESC;
```

### 應用層

```go
// 使用 context timeout 避免請求阻塞
ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
defer cancel()

// 使用 connection pool
pool, err := pgxpool.New(ctx, connectionString)

// 批次查詢優化
// Bad: N+1 queries
for _, memberID := range memberIDs {
    member, _ := repo.GetMemberByID(memberID)
}

// Good: Single batch query
members, _ := repo.GetMembersByIDs(memberIDs)
```

---

## 相關資源

### 文件

- [Feature Specification](./spec.md)
- [Implementation Plan](./plan.md)
- [Data Model](./data-model.md)
- [API Contracts](./contracts/openapi.yaml)
- [Research Notes](./research.md)

### 外部資源

- [Effective Go](https://go.dev/doc/effective_go)
- [Gin Documentation](https://gin-gonic.com/docs/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Testify Documentation](https://github.com/stretchr/testify)

### 支援

- **技術問題**：請在專案中開 Issue
- **緊急支援**：聯繫團隊 Slack #duotify-membership
- **程式碼審查**：使用 GitHub Pull Request

---

## 下一步

完成本地開發環境設定後，建議按以下順序進行：

1. **熟悉程式碼結構**：閱讀 `internal/` 目錄下的主要模組
2. **執行測試套件**：確保所有測試通過
3. **實作新功能**：參考 [tasks.md](./tasks.md) 任務清單
4. **提交程式碼**：遵循 Git commit 規範與 PR 流程

祝開發順利！🚀
