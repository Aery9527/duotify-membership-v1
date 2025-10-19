# Data Model - 會員註冊流程

**Feature**: 001-member-registration  
**Date**: 2025-10-19

本文件定義會員註冊與驗證功能的資料模型，包含實體定義、關聯關係、驗證規則與索引策略。

---

## 實體概覽

本功能包含三個核心實體：

1. **Member (會員帳戶)**: 儲存使用者基本資訊與帳號狀態
2. **VerificationCode (驗證碼)**: 管理 E-Mail 驗證碼生命週期
3. **VerificationResendLog (驗證碼重發記錄)**: 追蹤重發歷史實施速率限制

---

## 1. Member (會員帳戶)

### 用途
儲存註冊使用者的基本資料、認證資訊與帳號狀態。

### 欄位定義

| 欄位名稱 | 資料型別 | 約束 | 說明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | 會員唯一識別碼（使用 UUID v4） |
| national_id | VARCHAR(10) | UNIQUE, NOT NULL | 台灣身分證字號（1 英文 + 9 數字） |
| name | VARCHAR(100) | NOT NULL | 姓名（僅中英文字母，無空格符號） |
| email | VARCHAR(255) | NOT NULL, INDEX | 電子郵件地址 |
| password_hash | VARCHAR(255) | NOT NULL | Bcrypt 雜湊後的密碼 |
| is_verified | BOOLEAN | DEFAULT FALSE | E-Mail 驗證狀態 |
| is_locked | BOOLEAN | DEFAULT FALSE | 帳號鎖定狀態（驗證碼錯誤過多） |
| locked_until | TIMESTAMP | NULLABLE | 鎖定到期時間（NULL 表示未鎖定） |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 建立時間 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 最後更新時間 |

### 驗證規則

- **national_id**: 
  - 格式：`^[A-Z][0-9]{9}$`
  - 含檢查碼驗證（台灣身分證字號演算法）
  - 唯一性由資料庫 UNIQUE constraint 保證
  
- **name**: 
  - 格式：`^[\u4e00-\u9fa5a-zA-Z]+$` (僅中文或英文字母)
  - 長度：1-100 字元
  
- **email**: 
  - 格式：RFC 5322 標準 E-Mail 格式
  - 長度：最多 255 字元
  
- **password_hash**: 
  - Bcrypt 雜湊結果（原始密碼 8-20 碼，含大小寫英文與數字）
  - 儲存格式：`$2a$12$...` (Bcrypt with cost factor 12)

### 索引策略

```sql
-- 主鍵索引（自動建立）
PRIMARY KEY (id)

-- 唯一索引（確保身分證字號唯一性，處理併發註冊）
CREATE UNIQUE INDEX idx_member_national_id ON members(national_id);

-- 一般索引（支援 E-Mail 查詢，用於登入與驗證流程）
CREATE INDEX idx_member_email ON members(email);

-- 複合索引（支援鎖定狀態查詢）
CREATE INDEX idx_member_locked ON members(is_locked, locked_until) WHERE is_locked = TRUE;
```

### 狀態轉換

```
[註冊] → is_verified=FALSE, is_locked=FALSE
   ↓
[驗證成功] → is_verified=TRUE
   ↓
[錯誤 3 次] → is_locked=TRUE, locked_until=NOW()+10分鐘
   ↓
[10 分鐘後] → is_locked=FALSE, locked_until=NULL（自動或手動清理）
```

---

## 2. VerificationCode (驗證碼)

### 用途
管理 E-Mail 驗證碼的生成、過期與使用狀態。

### 欄位定義

| 欄位名稱 | 資料型別 | 約束 | 說明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | 驗證碼記錄唯一識別碼 |
| member_id | UUID | FOREIGN KEY, NOT NULL | 關聯的會員 ID |
| code | CHAR(6) | NOT NULL | 6 位數驗證碼（數字） |
| status | VARCHAR(20) | NOT NULL | 狀態：unused/used/expired/invalidated |
| attempt_count | INT | DEFAULT 0 | 錯誤嘗試次數 |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 產生時間 |
| expires_at | TIMESTAMP | NOT NULL | 過期時間（產生時間 + 5 分鐘） |
| used_at | TIMESTAMP | NULLABLE | 使用時間（驗證成功時記錄） |

### 驗證規則

- **code**: 
  - 格式：6 位數字字串 `[0-9]{6}`
  - 生成：使用 crypto/rand 產生隨機數字（0-999999，前置補 0）
  
- **status**: 
  - 枚舉值：`unused`, `used`, `expired`, `invalidated`
  - `unused`: 尚未使用且未過期
  - `used`: 已成功驗證
  - `expired`: 已過期（超過 5 分鐘）
  - `invalidated`: 因重發新驗證碼而失效
  
- **expires_at**: 
  - 計算：`created_at + INTERVAL '5 minutes'`
  - 驗證時必須檢查 `NOW() < expires_at`

### 索引策略

```sql
-- 主鍵索引
PRIMARY KEY (id)

-- 外鍵索引（快速查詢特定會員的驗證碼）
CREATE INDEX idx_verification_member_id ON verification_codes(member_id);

-- 複合索引（支援查詢最新的有效驗證碼）
CREATE INDEX idx_verification_lookup ON verification_codes(member_id, status, created_at DESC);

-- 清理索引（支援定期清理過期記錄）
CREATE INDEX idx_verification_cleanup ON verification_codes(expires_at) WHERE status = 'unused';
```

### 外鍵關聯

```sql
FOREIGN KEY (member_id) REFERENCES members(id) ON DELETE CASCADE
```
會員刪除時自動清理關聯的驗證碼記錄。

### 業務邏輯

**產生新驗證碼時**：
1. 將該會員所有 `status='unused'` 的舊驗證碼更新為 `status='invalidated'`
2. 插入新的驗證碼記錄（status='unused', expires_at=NOW()+5分鐘）

**驗證碼檢查時**：
1. 查詢該會員最新的 `status='unused'` 驗證碼
2. 檢查是否過期（NOW() < expires_at）
3. 比對驗證碼是否正確
4. 錯誤時遞增 attempt_count
5. 正確時更新 status='used', used_at=NOW()

---

## 3. VerificationResendLog (驗證碼重發記錄)

### 用途
記錄驗證碼重發歷史，實施每小時最多 3 次的速率限制。

### 欄位定義

| 欄位名稱 | 資料型別 | 約束 | 說明 |
|---------|---------|------|------|
| id | UUID | PRIMARY KEY | 記錄唯一識別碼 |
| member_id | UUID | FOREIGN KEY, NOT NULL | 關聯的會員 ID |
| resent_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 重發時間戳記 |

### 索引策略

```sql
-- 主鍵索引
PRIMARY KEY (id)

-- 複合索引（支援速率限制查詢：最近 1 小時內的重發次數）
CREATE INDEX idx_resend_rate_limit ON verification_resend_logs(member_id, resent_at DESC);
```

### 外鍵關聯

```sql
FOREIGN KEY (member_id) REFERENCES members(id) ON DELETE CASCADE
```

### 業務邏輯

**重發驗證碼前檢查**：
```sql
SELECT COUNT(*) FROM verification_resend_logs
WHERE member_id = ? AND resent_at > NOW() - INTERVAL '1 hour';
```
若計數 >= 3，拒絕重發並回傳錯誤訊息。

**清理策略**：
定期清理 1 小時以上的舊記錄（或保留作為審計日誌）。

---

## 實體關聯圖（ER Diagram）

```
┌─────────────────────────┐
│       Member            │
│─────────────────────────│
│ id (PK)                 │
│ national_id (UNIQUE)    │
│ name                    │
│ email                   │
│ password_hash           │
│ is_verified             │
│ is_locked               │
│ locked_until            │
│ created_at              │
│ updated_at              │
└────────┬────────────────┘
         │
         │ 1:N
         │
         ├─────────────────────────────────┐
         │                                 │
         ↓                                 ↓
┌────────────────────────┐   ┌─────────────────────────────┐
│  VerificationCode      │   │ VerificationResendLog       │
│────────────────────────│   │─────────────────────────────│
│ id (PK)                │   │ id (PK)                     │
│ member_id (FK)         │   │ member_id (FK)              │
│ code                   │   │ resent_at                   │
│ status                 │   └─────────────────────────────┘
│ attempt_count          │
│ created_at             │
│ expires_at             │
│ used_at                │
└────────────────────────┘
```

---

## 資料庫遷移腳本

### Migration: 001_create_members_and_verifications.sql

```sql
-- 建立 members 資料表
CREATE TABLE IF NOT EXISTS members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    national_id VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,
    is_locked BOOLEAN NOT NULL DEFAULT FALSE,
    locked_until TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 建立索引
CREATE UNIQUE INDEX idx_member_national_id ON members(national_id);
CREATE INDEX idx_member_email ON members(email);
CREATE INDEX idx_member_locked ON members(is_locked, locked_until) WHERE is_locked = TRUE;

-- 建立 verification_codes 資料表
CREATE TABLE IF NOT EXISTS verification_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL,
    code CHAR(6) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'unused',
    attempt_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    used_at TIMESTAMP,
    FOREIGN KEY (member_id) REFERENCES members(id) ON DELETE CASCADE
);

-- 建立索引
CREATE INDEX idx_verification_member_id ON verification_codes(member_id);
CREATE INDEX idx_verification_lookup ON verification_codes(member_id, status, created_at DESC);
CREATE INDEX idx_verification_cleanup ON verification_codes(expires_at) WHERE status = 'unused';

-- 建立 verification_resend_logs 資料表
CREATE TABLE IF NOT EXISTS verification_resend_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL,
    resent_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (member_id) REFERENCES members(id) ON DELETE CASCADE
);

-- 建立索引
CREATE INDEX idx_resend_rate_limit ON verification_resend_logs(member_id, resent_at DESC);

-- 建立 updated_at 自動更新觸發器
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_members_updated_at BEFORE UPDATE ON members
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- 新增註解
COMMENT ON TABLE members IS '會員帳戶資料表';
COMMENT ON TABLE verification_codes IS 'E-Mail 驗證碼資料表';
COMMENT ON TABLE verification_resend_logs IS '驗證碼重發記錄資料表';
```

---

## Go 結構體對應

### Member Entity

```go
package domain

import (
    "time"
    "github.com/google/uuid"
)

type Member struct {
    ID           uuid.UUID  `db:"id" json:"id"`
    NationalID   string     `db:"national_id" json:"national_id" validate:"required,taiwanid"`
    Name         string     `db:"name" json:"name" validate:"required,chinesename,max=100"`
    Email        string     `db:"email" json:"email" validate:"required,email,max=255"`
    PasswordHash string     `db:"password_hash" json:"-"`
    IsVerified   bool       `db:"is_verified" json:"is_verified"`
    IsLocked     bool       `db:"is_locked" json:"is_locked"`
    LockedUntil  *time.Time `db:"locked_until" json:"locked_until,omitempty"`
    CreatedAt    time.Time  `db:"created_at" json:"created_at"`
    UpdatedAt    time.Time  `db:"updated_at" json:"updated_at"`
}

// SetPassword hashes and sets the password
func (m *Member) SetPassword(plainPassword string) error {
    hash, err := bcrypt.GenerateFromPassword([]byte(plainPassword), 12)
    if err != nil {
        return err
    }
    m.PasswordHash = string(hash)
    return nil
}

// CheckPassword verifies the password
func (m *Member) CheckPassword(plainPassword string) error {
    return bcrypt.CompareHashAndPassword([]byte(m.PasswordHash), []byte(plainPassword))
}

// Lock locks the account for the specified duration
func (m *Member) Lock(duration time.Duration) {
    m.IsLocked = true
    lockedUntil := time.Now().Add(duration)
    m.LockedUntil = &lockedUntil
}

// Unlock unlocks the account
func (m *Member) Unlock() {
    m.IsLocked = false
    m.LockedUntil = nil
}

// IsCurrentlyLocked checks if the account is still locked
func (m *Member) IsCurrentlyLocked() bool {
    if !m.IsLocked {
        return false
    }
    if m.LockedUntil != nil && time.Now().After(*m.LockedUntil) {
        return false // Lock expired
    }
    return true
}
```

### VerificationCode Entity

```go
package domain

import (
    "time"
    "github.com/google/uuid"
)

type VerificationCodeStatus string

const (
    StatusUnused      VerificationCodeStatus = "unused"
    StatusUsed        VerificationCodeStatus = "used"
    StatusExpired     VerificationCodeStatus = "expired"
    StatusInvalidated VerificationCodeStatus = "invalidated"
)

type VerificationCode struct {
    ID           uuid.UUID                `db:"id" json:"id"`
    MemberID     uuid.UUID                `db:"member_id" json:"member_id"`
    Code         string                   `db:"code" json:"code" validate:"required,len=6,numeric"`
    Status       VerificationCodeStatus   `db:"status" json:"status"`
    AttemptCount int                      `db:"attempt_count" json:"attempt_count"`
    CreatedAt    time.Time                `db:"created_at" json:"created_at"`
    ExpiresAt    time.Time                `db:"expires_at" json:"expires_at"`
    UsedAt       *time.Time               `db:"used_at" json:"used_at,omitempty"`
}

// IsExpired checks if the verification code has expired
func (v *VerificationCode) IsExpired() bool {
    return time.Now().After(v.ExpiresAt)
}

// IsValid checks if the code can be used for verification
func (v *VerificationCode) IsValid() bool {
    return v.Status == StatusUnused && !v.IsExpired()
}

// IncrementAttempt increments the failed attempt counter
func (v *VerificationCode) IncrementAttempt() {
    v.AttemptCount++
}

// MarkAsUsed marks the code as successfully used
func (v *VerificationCode) MarkAsUsed() {
    v.Status = StatusUsed
    now := time.Now()
    v.UsedAt = &now
}

// Invalidate marks the code as invalidated (e.g., when a new code is generated)
func (v *VerificationCode) Invalidate() {
    v.Status = StatusInvalidated
}
```

### VerificationResendLog Entity

```go
package domain

import (
    "time"
    "github.com/google/uuid"
)

type VerificationResendLog struct {
    ID       uuid.UUID `db:"id" json:"id"`
    MemberID uuid.UUID `db:"member_id" json:"member_id"`
    ResentAt time.Time `db:"resent_at" json:"resent_at"`
}
```

---

## 資料完整性保證

### 併發控制

1. **身分證字號唯一性**：
   - 資料庫 UNIQUE constraint 作為最終防線
   - 應用層先檢查存在性（SELECT）再插入（INSERT）
   - 捕獲資料庫唯一約束違反錯誤並轉換為友善訊息

2. **驗證碼失效機制**：
   - 使用交易（Transaction）確保舊驗證碼失效與新驗證碼產生的原子性
   - 查詢時使用 `FOR UPDATE` 鎖定記錄避免競爭

### 資料清理策略

1. **過期驗證碼清理**：
   - 定期任務（Cron job）清理 1 天前的已過期驗證碼
   - 或保留用於審計，僅清理 30 天以上的記錄

2. **重發記錄清理**：
   - 清理 24 小時以上的重發記錄（速率限制僅需 1 小時）
   - 或保留作為使用統計分析資料

3. **帳號鎖定自動解除**：
   - 選項 A：定期任務檢查 `locked_until < NOW()` 並更新狀態
   - 選項 B：查詢時動態判斷（呼叫 `IsCurrentlyLocked()`）

---

## 效能考量

### 查詢最佳化

1. **註冊時身分證檢查**：
   ```sql
   SELECT EXISTS(SELECT 1 FROM members WHERE national_id = ?) 
   -- 使用 EXISTS 比 COUNT(*) 更高效
   ```

2. **驗證碼查詢**：
   ```sql
   SELECT * FROM verification_codes
   WHERE member_id = ? AND status = 'unused'
   ORDER BY created_at DESC LIMIT 1;
   -- 使用複合索引 idx_verification_lookup 快速定位
   ```

3. **速率限制檢查**：
   ```sql
   SELECT COUNT(*) FROM verification_resend_logs
   WHERE member_id = ? AND resent_at > NOW() - INTERVAL '1 hour';
   -- 使用 idx_resend_rate_limit 索引
   ```

### 連線池設定

```go
// PostgreSQL 連線池配置
config.MaxConns = runtime.NumCPU() * 2  // 最大連線數
config.MinConns = 2                      // 最小連線數
config.MaxConnLifetime = 1 * time.Hour   // 連線最大存活時間
config.MaxConnIdleTime = 10 * time.Minute // 閒置連線超時
config.HealthCheckPeriod = 1 * time.Minute // 健康檢查週期
```

---

## 安全性考量

### 敏感資料保護

1. **密碼**: 
   - 明文密碼絕不儲存或記錄到日誌
   - 使用 Bcrypt (cost=12) 雜湊
   - JSON 序列化時排除 `password_hash` 欄位（使用 `json:"-"` tag）

2. **身分證字號**: 
   - 敏感個資，記錄日誌時需遮罩（例如：A123****89）
   - 符合台灣個資法要求

3. **驗證碼**: 
   - 使用 crypto/rand 產生（非 math/rand）
   - 日誌中不顯示完整驗證碼

### SQL 注入防護

- 所有資料庫查詢使用參數化查詢（prepared statements）
- pgx 驅動自動處理參數逸出

---

## 測試資料

### 單元測試用假資料

```go
// 測試用會員資料
var TestMember = &Member{
    ID:         uuid.MustParse("123e4567-e89b-12d3-a456-426614174000"),
    NationalID: "A123456789",
    Name:       "測試使用者",
    Email:      "test@example.com",
    IsVerified: false,
    IsLocked:   false,
    CreatedAt:  time.Now(),
    UpdatedAt:  time.Now(),
}

// 測試用驗證碼
var TestVerificationCode = &VerificationCode{
    ID:        uuid.New(),
    MemberID:  TestMember.ID,
    Code:      "123456",
    Status:    StatusUnused,
    CreatedAt: time.Now(),
    ExpiresAt: time.Now().Add(5 * time.Minute),
}
```

---

## 總結

資料模型設計完成，涵蓋：
- ✅ 三個核心實體與完整欄位定義
- ✅ 關聯關係與外鍵約束
- ✅ 索引策略確保查詢效能
- ✅ 驗證規則符合規格書要求
- ✅ Go 結構體對應與業務邏輯方法
- ✅ 資料庫遷移腳本
- ✅ 併發控制與資料完整性保證
- ✅ 效能最佳化與安全性考量

可以進入 API 合約設計階段。
