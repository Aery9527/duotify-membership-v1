# Phase 0: Research - 會員註冊流程

**Feature**: 001-member-registration  
**Date**: 2025-10-19

本文件記錄技術選型研究結果，解決實作計畫中的 NEEDS CLARIFICATION 項目。

## 研究項目總覽

從 Technical Context 中識別出以下需要研究的項目：

1. 資料庫選擇（PostgreSQL/MySQL/其他）
2. 資料庫驅動函式庫
3. E-Mail 發送服務
4. 密碼加密函式庫
5. 輸入驗證函式庫
6. 測試框架選擇

---

## 1. 資料庫選擇

### Decision
**PostgreSQL 14+**

### Rationale
- **ACID 交易支援**：確保註冊流程的資料一致性，特別是處理併發註冊時的身分證字號唯一性約束
- **UNIQUE 索引效能**：PostgreSQL 的 UNIQUE constraint 實作高效，支援部分索引（partial index）可優化查詢效能
- **JSON 支援**：內建 JSONB 型別，未來若需儲存額外的會員屬性或審計記錄時具彈性
- **成熟的 Go 生態系**：pgx 驅動提供優異效能與功能完整性
- **開源且活躍社群**：降低授權成本，豐富的文件與問題解決資源

### Alternatives Considered
- **MySQL 8.0**: 同樣支援 ACID 與 UNIQUE constraint，但 PostgreSQL 在併發控制（MVCC）與複雜查詢最佳化上表現更優
- **SQLite**: 適合嵌入式或開發環境，但不適合生產環境的高併發場景
- **MongoDB**: NoSQL 資料庫，雖彈性高但關聯式資料模型更適合會員註冊的強一致性需求

---

## 2. 資料庫驅動函式庫

### Decision
**pgx v5** (github.com/jackc/pgx/v5)

### Rationale
- **原生 PostgreSQL 驅動**：直接支援 PostgreSQL 特性，無需透過 database/sql 抽象層，效能最佳
- **批次操作支援**：提供 Batch API，適合未來批次查詢或更新場景
- **連線池管理**：內建 pgxpool，自動管理連線生命週期與資源釋放
- **型別安全**：強型別 API 減少執行期錯誤
- **活躍維護**：定期更新，社群活躍，Issue 回應快速

### Alternatives Considered
- **database/sql + pq**: 標準函式庫抽象層，但效能較 pgx 低約 20-30%，且缺少 PostgreSQL 特定功能
- **GORM**: ORM 框架雖然方便但增加抽象層複雜度，效能開銷較大，且 Constitution 要求保持程式碼簡潔

---

## 3. E-Mail 發送服務

### Decision
**SMTP 直接發送（使用 Go 標準 net/smtp）+ 第三方 SMTP 服務（如 SendGrid/AWS SES）**

### Rationale
- **標準函式庫支援**：Go 內建 net/smtp，無需額外依賴，降低複雜度
- **靈活性**：可輕鬆切換 SMTP 服務提供商（SendGrid, AWS SES, Mailgun）
- **可測試性**：易於 mock SMTP 連線進行單元測試與合約測試
- **成本效益**：多數 SMTP 服務提供免費額度（例如 SendGrid 每日 100 封），適合初期規模

### Implementation Notes
- 使用環境變數配置 SMTP 主機、埠、認證資訊
- 實作重試機制（exponential backoff）處理暫時性失敗
- 記錄發送失敗的審計日誌供事後追查
- E-Mail 模板使用 Go html/template 確保內容安全（防止注入攻擊）

### Alternatives Considered
- **第三方 SDK（如 SendGrid Go client）**: 功能豐富但綁定特定服務商，遷移成本高
- **自建 SMTP 伺服器**: 維運成本高，需處理黑名單、送達率等複雜問題，不適合初期階段

---

## 4. 密碼加密函式庫

### Decision
**bcrypt** (golang.org/x/crypto/bcrypt)

### Rationale
- **業界標準**：Bcrypt 是密碼雜湊的事實標準，經過時間考驗
- **自動加鹽**：內建隨機鹽值生成，每次雜湊結果不同，防止彩虹表攻擊
- **可調整成本因子**：可配置工作因子（cost factor），隨硬體進步調整，維持 ~100-200ms 運算時間
- **Go 官方擴展包**：golang.org/x/crypto 由 Go 團隊維護，穩定可靠
- **憲法合規**：符合 Constitution 要求的業界標準加密演算法

### Implementation Notes
- 使用 cost factor = 12 (約 200ms 運算時間，符合憲法 <200ms p95 要求)
- 密碼驗證在服務層進行，不在資料庫層處理
- 明文密碼絕不記錄到日誌

### Alternatives Considered
- **Argon2**: 較新的演算法，記憶體密集型，防禦 ASIC 攻擊更好，但 Bcrypt 已足夠安全且更成熟
- **scrypt**: 記憶體密集型，但社群支援與文件不如 Bcrypt 豐富
- **PBKDF2**: 較舊的標準，NIST 推薦但迭代次數設定複雜，不如 Bcrypt 簡單

---

## 5. 輸入驗證函式庫

### Decision
**go-playground/validator v10** (github.com/go-playground/validator/v10)

### Rationale
- **宣告式驗證**：使用 struct tags 定義驗證規則，程式碼簡潔易讀
- **豐富的內建驗證器**：支援 required, email, min, max, regex 等常見驗證
- **自訂驗證器支援**：可擴展自訂台灣身分證字號驗證邏輯
- **本地化錯誤訊息**：支援多語系錯誤訊息，符合憲法的繁體中文要求
- **Gin 整合良好**：Gin 框架預設綁定此函式庫，減少整合工作

### Implementation Notes
- 實作自訂驗證器 `taiwanid` 用於身分證字號格式檢查（含檢查碼驗證）
- 實作自訂驗證器 `chinesename` 用於姓名格式檢查（僅中英文字母）
- 實作自訂驗證器 `strongpassword` 用於密碼強度檢查（8-20 碼含大小寫與數字）
- 所有錯誤訊息使用繁體中文，符合憲法要求

### Alternatives Considered
- **手動驗證**: 靈活但程式碼冗長，難以維護，測試覆蓋困難
- **ozzo-validation**: API 流暢但社群較小，文件不如 go-playground 豐富

---

## 6. 測試框架選擇

### Decision
**Go 標準 testing + testify/assert + testify/mock**

### Rationale
- **零依賴核心**：Go 內建 testing 套件已提供完整測試功能（測試執行、基準測試、子測試）
- **testify/assert 增強可讀性**：提供清晰的斷言語法與詳細的失敗訊息，符合憲法測試品質要求
- **testify/mock 簡化 mocking**：輕量級 mock 框架，適合隔離外部依賴（資料庫、SMTP）
- **Table-driven tests 支援**：Go 慣用模式，適合大量測試案例（規格書定義多個驗收情境）
- **效能追蹤內建**：Go benchmark 功能符合憲法效能測試要求

### Test Organization
```
tests/
├── unit/                   # 單元測試（業務邏輯、驗證器）
│   ├── validator_test.go
│   ├── service_test.go
│   └── crypto_test.go
├── integration/            # 整合測試（API + 資料庫）
│   ├── registration_test.go
│   ├── verification_test.go
│   └── testdata/
└── contract/               # 合約測試（E-Mail 服務）
    └── email_service_test.go
```

### Alternatives Considered
- **Ginkgo/Gomega**: BDD 風格框架，功能豐富但增加學習曲線與依賴複雜度
- **GoConvey**: 提供 Web UI，但標準工具鏈已足夠且更符合 Go 慣例

---

## 額外研究：最佳實踐

### Go 1.25.0 新特性應用
- **範圍迭代器（range over func）**: 可用於批次處理驗證碼清理任務
- **改進的型別推斷**: 減少冗餘型別宣告，提升程式碼可讀性
- **標準函式庫增強**: 使用最新的 context 與 slog 套件

### Gin 框架最佳實踐
- **中介軟體鏈**: recovery → logger → validator → auth
- **錯誤處理**: 使用 Gin 的 `c.Error()` + 全域錯誤處理中介軟體統一回應格式
- **Context 傳遞**: 將 request ID 注入 context 供 logging 使用
- **優雅關閉**: 實作 graceful shutdown 確保請求處理完畢

### 併發安全模式
- 資料庫層依賴 UNIQUE constraint 作為最終防線
- 應用層使用樂觀檢查（SELECT then INSERT）提供友善錯誤訊息
- 避免分散式鎖複雜度（PostgreSQL ACID 已足夠）

### 效能最佳化策略
- 資料庫連線池設定：max_conns = CPU 核心數 * 2
- 準備語句（prepared statements）減少解析開銷
- 索引策略：UNIQUE 索引於 national_id 欄位，B-tree 索引於 email 欄位
- Context timeout：API 層 5s，資料庫查詢層 2s

---

## 研究總結

所有 NEEDS CLARIFICATION 項目已解決，技術選型決策完成：

| 項目 | 決策 | 狀態 |
|------|------|------|
| 資料庫 | PostgreSQL 14+ | ✅ Resolved |
| 資料庫驅動 | pgx v5 | ✅ Resolved |
| E-Mail 服務 | net/smtp + SMTP provider | ✅ Resolved |
| 密碼加密 | bcrypt (cost=12) | ✅ Resolved |
| 輸入驗證 | go-playground/validator v10 | ✅ Resolved |
| 測試框架 | testing + testify | ✅ Resolved |

所有選擇符合 Constitution 原則：
- 使用 Go 慣用模式與標準函式庫優先
- 保持依賴最小化
- 效能目標可達成（<200ms p95）
- 測試策略完整（80%+ 覆蓋率）
- 安全性符合業界標準

可以進入 Phase 1 設計階段。
