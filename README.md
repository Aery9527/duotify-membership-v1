# Spec Kit 使用練習

References:

- [github: Spec Kit](https://github.com/github/spec-kit)
- [github: 保哥 Spec Kit 翻譯](https://github.com/doggy8088/spec-kit)
- [多奇教育: SDD with Spec Kit](https://learn.duotify.com/courses/sdd)

---

### 安裝 Spec Kit

1. 使用 PowerShell 5 + (win+R -> `powershell`)
    ```
    # 安裝 Scoop 套件管理器
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
    Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
    
    # 安裝 pwsh、git、uv 與 nvm
    scoop install pwsh git uv nvm
    
    # 安裝搜尋文字檔案的利器 ripgrep
    scoop install ripgrep
    
    # 安裝 Windows Terminal
    scoop bucket add extras
    scoop install extras/windows-terminal
    
    # 更新 uv 的路徑
    uv tool update-shell
    
    # 重新啟動檔案總管，讓環境變數生效
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```
2. 開啟 Windows Terminal 終端機視窗 (win+R -> `wt`)
   ```
    # 透過 nvm 安裝 Node.js LTS 版本
    nvm install lts
    nvm use lts
    
    # 安裝 Spec Kit 的 Specify CLI 命令列工具
    uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
    
    # 安裝 Copilot CLI (請自由選擇)
    npm install -g @github/copilot
    
    # 安裝 Claude Code (請自由選擇)
    npm install -g @anthropic-ai/claude-code
    
    # 檢查 Spec Kit 需要的工具支援
    specify check
   ```
3. 至少需要看到 Git version control 是綠燈, 還有 GitHub Copilot 或 Claude Code 或任何一套 AI Coding 工具是綠燈.
4. 升級 `uv tool install specify-cli --force --from git+https://github.com/github/spec-kit.git`

---

## START

1. 在專案根目錄初始化 Spec Kit 專案範本 `specify init --ai copilot --script ps --here`
2. 建立一個 `AGENTS.md` 檔案 (給 Copilot CLI 看的), 內容: We're going to be using slash command from `.github\prompts\`
3. `git add AGENTS.md && git commit -m "Add AGENTS.md"`
4. 更新 Copilot CLI 到最新版 `npm i -g @github/copilot`
5. 查看 Copilot CLI 版本 `copilot -v`
6. 啟動 GitHub Copilot CLI 互動式環境 `copilot --allow-all-tools`
7. *Spec Kit Step 1*: 制定憲法 (也可以指定檔案讀取內容)
    - `/speckit.constitution Create principles focused on code quality, testing standards, user experience consistency, and performance requirements`
    - `/speckit.constitution All specifications, plans, and user-facing documentation MUST be written in Traditional Chinese (zh-TW)`
8. *Spec Kit Step 2*: 制定規格 `/speckit.specify PRD.md`, 會讀取 PRD.md 內容理解 **User Story**
    - 隨時可以再次執行修正, 支援差異更新
    - 這步驟會自動 fork branch, 然後根據 prd 建立規格 [spec.md](specs/001-member-registration/spec.md)
9. *Spec Kit Step 3*: 釐清規格 `/speckit.clarify`
    - 這步是讓 AI 根據 [spec.md](specs/001-member-registration/spec.md) 內容釐清模糊的部分, 就像 SA 在需求訪談一樣
    - AI 會有初步的選項讓你選擇, 也可以自己輸入問題答案, 可以多次執行該指令釐清需求問題
10. *Spec Kit Step 4*: 制定開發計畫 `/speckit.plan @tech.md`
11. *Spec Kit Step 5*: 規劃任務清單 `/speckit.tasks`
12. *Spec Kit Step 6*: 產生規格分析報告 `/speckit.analyze Save your analyze report to analyze-01.md`
13. *Spec Kit Step 7*: 開始實作 `/speckit.implement`